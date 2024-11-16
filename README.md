# zinc
Home of the Zinc Programming language

```
// In general, a definition goes: <var-type> <var-name> = <expression>
// however you can infer the type: <var-name> := <expression>
// there is

// for reference:
// i32 i = 0
// fn name = fn(parameters) {}
// type name = type(generic-args) {}
// is equivelent to
// i := 0
// name := fn(parameters) {}
// name := type(generic-args) {}

// functions and types have added equivelent syntax:
// fn name(parameters) {}
// type name(generic-args) {}

// since zinc standard library is largely incomplete, we can use the libc wrapper for I/O
io := #import("libc.stdio")

// a '#' signifies this function can only run at compile time
// everything in the global scope is calculated at compile time

// here #import is a compile time function that resolves to a type
// compile time functions will be JIT compiled and run during compile-time using MIR
// this will give the language full capability to do any pre-processing necessary
// most of the #import logic can be written entirely in zinc as a result within the standard library
// and it won't have to be baked into the compiler
// by default, functions from the main "zinc" package will be visible such as
// - primitive types
// - #import([]u8 package) type       imports a package
// - #meta(type A) metadata           gets compile-time metadata for a type (structure layout / added metadata)
// - #impl(type A, type B) !void      creates an implemenation of a given interface A from a type B
// - cast(type A, (type B) value) A   casts value to type A

// this creates a printable interface
// in zinc, an interface is just a set of function pointers
// this is simple, doesn't require new language features or syntax, and fully compatable with C ABI

type printable(T) {
    ^fn(T self) []u8 to_str
}

// by default, the type gets metadata describing it's size
// #assert(printable(i32).size == ^void.size)

// instance functions are simply metadata attached to the type
// printable(T).foo := fn(self) {}

// a function to cast from one type to another
fn cast(type A, (type B) value) A {
    // alternatively: union := type { A a | B b }
    // nested types and functions are allowed
    type union { A a | B b }
    ret := union { b = value }
    return ret.a
}

// function to automate interface implementation!
fn #impl(type A, type B) !void {
    tb := #meta(B)
    ta := #meta(A(B))
    data := A(B) {}
    raw := cast(^void, @data)
    for(field in ta.fields) {
        metafunc := try tb.meta.find(field, fn(a, b) -> streq(a.name, b.name))
                    or #err("Missing function % for implementation of %", .{ field.name, ta.name })
        cast(^field.type, @raw[field.offset])^ = metafunc.value
    }
    tb.data.add(ta.name, data)
}

type person {
    []u8 first_name;
    []u8 last_name;
}

fn person.to_str(self) []u8 {
    ret := []u8.init(self.first_name.len + self.last_name.len + 1)
    memcpy(ret, self.first_name)
    ret[self.first_name.len] = ' '
    memcpy(ret + self.first_name.len + 1, self.last_name)
    return ret
}

#impl(printable, person)
// this is the equivalent: person.printable := printable(person) { @person.to_str }

// function accepting an printable interface
// default arguments are type-checked at the function call,
// so the call will error if type T doesn't have an implementation of the printable interface
fn print(^$T self, printable(T) vtable = T.printable) {
    vtable.to_str(self)
}

jim := person { "Jim", "Jones" }

print(@jim)

// this is a function to cast.
// @void ptr = malloc(8)
// str := cast(@u8, ptr)

// add a foreach method to arrays of type T
fn [](type T).foreach(self, fn(T) apply) {
    for (i := 0; i < self.len; i++) -> apply(self[i])
}

// array of addresses to unsigned 8-bit integers (characters)
fn main([]@u8 args) {
    args.foreach(fn(a) -> io.printf("%s", .{ a }))
}
```
