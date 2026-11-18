# Abilities API updates in WordPress 7.2

> Review draft, checked against WordPress trunk `19379c816819a77e55a2f2f577ac9809340cc407` on November 18, 2026. The capabilities section describes the current implementation. [#66325](https://core.trac.wordpress.org/ticket/66325#comment:21) was reopened after its November 17 closure; the release-scope discussion remains open.
>
> Consolidated from gziolo’s original notes: [JSON values](https://core.trac.wordpress.org/ticket/66267#comment:44), [required capabilities](https://core.trac.wordpress.org/ticket/66325#comment:8), and [input sanitization](https://core.trac.wordpress.org/ticket/66383#comment:26). JSON response coverage reflects the follow-up in [changeset 65072](https://core.trac.wordpress.org/changeset/65072).
>
> This review preface is not part of the proposed published post.

WordPress 7.2 updates the Abilities API to make permissions, input handling, and JSON output more predictable. Abilities can declare capabilities required for every call. Direct PHP calls and REST requests sanitize input according to the ability’s schema, and JSON results preserve the object or array shape that the schema declares. The examples below show what ability authors and integration developers need to change.

## Declare required capabilities

The new `required_capabilities` registration argument accepts a list of non-empty capability strings. During permission checking, WordPress checks each capability with `current_user_can()`. All listed capabilities must pass before the permission callback runs.

Use this list for requirements that apply to every invocation. For example, `core/settings-get` always requires `manage_options`. Its registration declares that capability and omits `permission_callback`. This excerpt shows its permission arguments:
```php
array(
	'required_capabilities' => array( 'manage_options' ),
	// No permission_callback: the capability covers every call.
);
```
When permissions also depend on the input, keep that check in `permission_callback`. The following ability requires `edit_posts` for every call, then checks whether the user can edit the requested post. The example assumes that the `my-plugin` category is already registered:
```php
add_action(
	'wp_abilities_api_init',
	static function () {
		wp_register_ability(
			'my-plugin/get-post-title',
			array(
				'label'                 => __( 'Get post title', 'my-plugin' ),
				'description'           => __( 'Read the title of a post you can edit.', 'my-plugin' ),
				'category'              => 'my-plugin',
				'required_capabilities' => array( 'edit_posts' ),
				'input_schema'          => array(
					'type'    => 'integer',
					'minimum' => 1,
				),
				'output_schema'         => array( 'type' => 'string' ),
				'permission_callback'   => static function ( int $post_id ): bool {
					return current_user_can( 'edit_post', $post_id );
				},
				'execute_callback'      => static function ( int $post_id ): string {
					return get_post_field( 'post_title', $post_id, 'raw' );
				},
				'meta'                  => array(
					'show_in_rest' => true,
					'annotations' => array( 'readonly' => true ),
				),
			)
		);
	}
);
```
Choose the permission arguments according to the ability’s requirements:

- If a non-empty `required_capabilities` list covers every permission check, omit `permission_callback`. You do not need `__return_true`.
- If a call needs additional checks, such as access to a particular post, provide a callback. The declared capabilities are checked first.
- If `required_capabilities` is empty or omitted, a valid `permission_callback` is required. Use `__return_true` only when the ability is intentionally executable by everyone; otherwise, provide the appropriate permission check.

The existing `wp_ability_permission_result` filter still receives the result and can override it, including a denial caused by a missing capability.

### Filter discovery by capabilities

Integrations can read the declaration with `$ability->get_required_capabilities()`. It is also included in REST discovery responses.

To request abilities whose declared requirements the current user meets:

```php
$abilities = wp_get_abilities(
	array( 'has_required_capabilities' => true )
);
```

The REST collection supports the same opt-in filter:

```text
GET /wp-json/wp-abilities/v1/abilities?has_required_capabilities=true
```

This filter does not run permission callbacks or the `wp_ability_permission_result` filter. Abilities without declared capabilities pass this filter (other discovery filters can still exclude them), and an ability that passes discovery can still deny execution for a particular input. In the example above, an author may discover the ability while lacking permission to read another author's draft through it.

Declare only the capabilities needed for every invocation. Putting an input-specific requirement in the static list can hide an ability from users who are allowed to execute it with other inputs.

Related ticket: [#66325](https://core.trac.wordpress.org/ticket/66325).

## Prepare input consistently across execution paths

`WP_Ability::prepare_input()` now handles the input preparation used by `execute()`:

1. Normalize the input, including applying a top-level schema default when the input is `null`.
2. Validate it against the input schema.
3. Sanitize the validated value using the schema.

The permission and execute callbacks receive the prepared value. For the example above, `$ability->execute( '123' )` passes the integer `123` to both callbacks. Schema formats also matter: a string declared with `format: text-field` has its tags stripped during sanitization.

Validation runs before sanitization. Invalid values are rejected before casting or property removal can hide the error. A failure during preparation returns a `WP_Error` and stops execution.

The REST execution controller prepares the original request input independently for its permission check and for execution. Custom integrations that call `execute()` can pass the raw input and let the ability prepare it.

### Number inputs become floats

A `number` schema converts an integer such as `10` to the float `10.0` in direct PHP calls, matching REST behavior. Use an `integer` schema when a callback needs integers. Callbacks using `number` should accept floats and avoid checks such as `is_int( $input )` or `$input === 10`.

Other schema conversions include the string `"false"` becoming the boolean `false`, and `"1,2"` becoming `array( 1, 2 )` for an array-of-integers schema.

### Sanitization failures stop execution

Input can pass validation and still fail sanitization. For example, `array( '1', '01' )` contains distinct strings, but both become the integer `1`. That fails an array-of-integers schema with `uniqueItems: true`.

In 7.2, this failure returns `ability_invalid_input`, including when it occurs in a nested field, before permission or execute callbacks run. The REST endpoint returns HTTP 400. It no longer falls back to the original unsanitized input when one field fails; such a fallback could otherwise leave a permission callback with the string `"false"`, which is truthy in PHP.

### Direct permission checks need prepared input

`check_permissions()` does not prepare its input. If an integration calls it directly, it must first call `prepare_input()` and stop on an error:

```php
$ability = wp_get_ability( 'my-plugin/get-post-title' );

if ( null === $ability ) {
	return new WP_Error( 'example_missing_ability', 'The ability is not registered.' );
}

$input = $ability->prepare_input( '123' );

if ( is_wp_error( $input ) ) {
	return $input;
}

// $input is now the integer 123.
$permission = $ability->check_permissions( $input );

if ( is_wp_error( $permission ) ) {
	return $permission;
}

if ( true !== $permission ) {
	return new WP_Error( 'example_forbidden', 'You cannot edit this post.' );
}

// This checks permission only. To run the ability, use execute().
```

A direct permission check does not execute the ability. For execution, call `execute()`; it performs preparation and checks permissions as part of the normal lifecycle.

Calling `normalize_input()` alone is insufficient: it does not include schema validation and sanitization. Likewise, `validate_input()` returns a validation result, not a prepared value.

A custom permission-check path that already normalizes and validates input can replace those steps with `prepare_input()`. Keep the error check before calling `check_permissions()`:
```diff
- $input = $ability->normalize_input( $raw_input );
+ $input = $ability->prepare_input( $raw_input );
  if ( is_wp_error( $input ) ) {
      return $input;
  }

- $valid = $ability->validate_input( $input );
- if ( is_wp_error( $valid ) ) {
-     return $valid;
- }

  $permission = $ability->check_permissions( $input );
```
This adds schema sanitization to the custom path. Continue to handle the permission result as shown above.
The REST controller's `sanitize_input_for_ability()` method is deprecated in 7.2 and is no longer registered as the input sanitizer. Its compatibility implementation returns the input unchanged.

Related ticket: [#66383](https://core.trac.wordpress.org/ticket/66383).

## Preserve empty objects in JSON

An empty PHP array encodes as `[]`, even when the schema describes an object. That can leave a client expecting an object with an array instead.

The shared `wp_prepare_json_schema_for_client()` function prepares schemas for use outside WordPress’s server-side validation. It removes unsupported keywords and prepares nested schemas and their existing defaults, so it is useful wherever a plugin exposes a schema to a REST client or frontend. In 7.2, its default-value handling preserves empty objects according to the schema:

```php
$schema = array(
	'type'    => 'object',
	'default' => array(),
);

echo wp_json_encode( wp_prepare_json_schema_for_client( $schema ) );
// {"type":"object","default":{}}
```

The new `wp_prepare_json_value_for_client()` function applies the same distinction to result values, including nested properties and array items:

```php
$schema = array(
	'type'       => 'object',
	'properties' => array(
		'settings' => array( 'type' => 'object' ),
		'items'    => array(
			'type'  => 'array',
			'items' => array( 'type' => 'object' ),
		),
	),
);

$result = array(
	'settings' => array(),
	'items'    => array( array() ),
);

echo wp_json_encode( wp_prepare_json_value_for_client( $result, $schema ) );
// {"settings":{},"items":[{}]}
```

The Abilities REST API applies these helpers when preparing schemas and execution results, including root empty objects and objects inside top-level lists. The WordPress AI Client also prepares ability results using the output schema. A direct PHP call to `$ability->execute()` still returns the callback's result without this JSON representation step. Custom transports that serialize the result should prepare it with the ability's output schema before encoding it.

This preparation does not perform full schema validation or fill in missing defaults. It changes the representation of existing values. An empty array stays `[]` when the matching schema permits an array; a schema that permits both objects and arrays does not force it to become `{}`.

### What integration developers should change

Ability authors should declare `type: object` or `type: array` explicitly in the output schema, including nested properties whose empty values matter. A client consuming an object schema should expect `{}` for an empty result. Update code that depended on receiving `[]` in that case. Results declared as arrays continue to use `[]`.
For a custom transport, prepare a successful result immediately before JSON encoding. Here, `$ability` is an already resolved `WP_Ability`, and `$raw_input` is the caller’s input:
```php
$result = $ability->execute( $raw_input );

if ( is_wp_error( $result ) ) {
	return $result;
}

return wp_json_encode(
	wp_prepare_json_value_for_client(
		$result,
		$ability->get_output_schema()
	)
);
```
The helper preserves the schema’s JSON shape; it does not grant permission or replace execution. Keep errors on the transport’s normal error path instead of preparing them as successful results.
Related ticket: [#66267](https://core.trac.wordpress.org/ticket/66267).

## Updating an integration

Review callbacks that depend on the original PHP type or unsanitized input. Check custom permission-check paths for missing preparation, and apply JSON preparation at serialization boundaries when using a custom transport.

Plugins supporting earlier WordPress versions must use version-appropriate registration arguments, retain their existing permission callbacks, and guard use of the new methods and functions. Omitting a permission callback in favor of `required_capabilities` requires WordPress 7.2.

Please test these changes against your abilities and share any compatibility concerns on the related tickets.
