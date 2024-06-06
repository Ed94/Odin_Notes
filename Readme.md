# Odin Notes

## Adding builtin types

Odin's language specification doesn't support operator overloading for non-user symbols. That doesn't stop the user from making a fork and adding builtin types as they desire.

(TODO: Not finished)
~~See : [compiler_adding_builtin_types.md](compiler_adding_builtin_types.md)~~

## Packages

### File name tricks

The compiler automatically omits specific files based on their names

* Files that start with a `.` are ignored.
* If the file has a Operating System or Architecture target that doesn't match the target for compilation.

```cpp
// Note this was pasted in 2024/6/6 (check the source file for latest, See `is_excluded_target_filename` in build_settings.cpp

gb_global String target_os_names[TargetOs_COUNT] = {
	str_lit(""),
	str_lit("windows"),
	str_lit("darwin"),
	str_lit("linux"),
	str_lit("essence"),
	str_lit("freebsd"),
	str_lit("openbsd"),
	str_lit("netbsd"),
	str_lit("haiku"),
	
	str_lit("wasi"),
	str_lit("js"),
	str_lit("orca"),

	str_lit("freestanding"),
};

gb_global String target_arch_names[TargetArch_COUNT] = {
	str_lit(""),
	str_lit("amd64"),
	str_lit("i386"),
	str_lit("arm32"),
	str_lit("arm64"),
	str_lit("wasm32"),
	str_lit("wasm64p32"),
};
```

### Non-cyclic

Packages must have their depedencies resolved downstream. This means at worst case until a proper generalization can be made, two definitions may need to be present between packages until a mutual depedent package can provide that dependency.

This can be alliviated (possibly, not tested pragmatically) without "monolithic packages feature fork" by having "package-pairs". Where there is a header and terminating package. The header package can container functionality or state information that would not generate cyclic dependencies. The terminating package would resolve in a cyclic depenedency if not used in a more "downstream" package.

### 1:1 With filesystem directories

Packages are tied to the filesystem by default.  
If you would like to be able to organize a package iresspective of the filesystem a fork is required from the offical compiler and `try_add_import_path` in `parser.cpp` would need to be adjusted *(as of: 2024/5/30)*.  
For support with ols (Odin Language Server), the changes are more rigourous as there are serveral places making the assumption that the package is the immeidate relative directory.

### Mapping Symbols for Definitions across packages (aliasing)

Any definitions not defined within its own package (foreign to the package) must have the package's implicit or defined name prefixed first and accessed with the member access operator: `.`, (`<package_associated_alias>).symbol`)

They can be alias with the following pattern:
`<desired_name_within_package> :: <package_associated_alias>.symbol`

This would be used with explicit procedure overload mappings to create succint procedure names if desried:

```odin
<desired_proc_name_mapped> :: proc {
  <package_associated_alias_a>.proc_symbol_a,
  <package_associated_alias_b>.proc_symbol_b,
  ...
}
```
(note: the make mapping can for example be extended or modified using this method )

This could be allievated by setting up a stage metaprogram that can auto alias commonly used symbols across packages of a codebase into a "mappings.odin" file.

## Managing a "persistent context"

This scenario would occur in other langauges as well this is just an example of some patterns that can be adopted when there's a need/want to have a persistent context of some kind for  a group of procedures in the package's interface.

### Unbounded Lifetime

For a context that will persist for the duration of a *module's* lifetime. Modules in this case considered to be runtime memory loaded by the OS for either a process or a dll (dyanmically loaded library) during runtime.

Have a pointer in the static data segment of the module(1.)

```odin
package <name>

PackageContextType :: struct {
    ...
}

@(private)
Module_Context : ^PackageContextType

set_module_context :: proc "contextless" ( context_to_assign : ^PackageContextType ) {
    Module_Context = context_to_assign
}

// ... Interface that relies on the context
```

The Module_Context must be set either during the first-load (or use) of the module's context. If the module is dynamically linked and then *hot-reloaded*, the module's context would need to be set again as its static data segment's state would have been lost (exceedingly rare case if not).

### Bound to a Scope's lifetime

The context is either "stackable" or some singular tracked instance. But its expected to not last beyond a specific "Scope" within execution. The scope is usually tied to a "block" scope. So long as the scope doesn't persist longer than the event where a hot-reload would occur. The context would not need to have a mechanism for explicitly setting, and instead could be embedded into a begin-end procedure set pattern for its interface.

```odin
ScopeContextType :: struct {
    ...
}

Some_Global_Container_To_Track_The_Scope/s : ^Container

some_scope_begin :: proc "contextless" ( relevant_ctx : ^ScopeContextType ) {
    ...
}

// ... Interface that relies on the the last pushed/set scope context

some_scope_end :: proc "contextless" ( relevant_ctx : ^ScopeContextType ) {
    ...
}

@(deferred_in = some_scope_end)
some_scope :: #force_inline proc "contextless" ( relevant_ctx : ^ScopeContextType ) { some_scope_begin(relevant_ctx) }
```

### Context explicitly passed always

This forms a situation where its up to the user to wrap the package's interface for their own use case, and the provided implementation requires the user to explicitly pass the context as a parameter of the procedures.

These are used throughout Odin's core & vendor collection's packages.  
See:

* `core:prof/spall`
* `vendor:fontstash`
* `vendor:microui`
* `vendor:nanovg`

To alleviate the constant explicit passign the following wrapper pattern may be used (example uses spall):

```odin
package <codebase>

import "core:prof/spall"

SpallProfilerContext :: struct {
	ctx     : spall.Context,
	buffer  : spall.Buffer,
}

profile_begin :: #force_inline proc "contextless" ( name : string, loc := #caller_location ) {
	spall._buffer_begin( & Codebase_Memory.profiler.ctx, & Codebase_Memory.profiler.buffer, name, "", loc )
}

profile_end :: #force_inline proc "contextless" () {
	spall._buffer_end( & Codebase_Memory.profiler.ctx, & Codebase_Memory.profiler.buffer)
}

@(deferred_none = profile_end) 
profile :: #force_inline proc "contextless" ( name : string, loc := #caller_location ) { profile_begin(name, loc) }
```

---
