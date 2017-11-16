[![Dub version](https://img.shields.io/dub/v/asdf.svg)](http://code.dlang.org/packages/asdf)
[![Dub downloads](https://img.shields.io/dub/dt/asdf.svg)](http://code.dlang.org/packages/asdf)
[![License](https://img.shields.io/dub/l/asdf.svg)](http://code.dlang.org/packages/asdf)
[![codecov.io](https://codecov.io/github/tamediadigital/asdf/coverage.svg?branch=master)](https://codecov.io/github/tamediadigital/asdf?branch=master)
[![Build Status](https://travis-ci.org/tamediadigital/asdf.svg?branch=master)](https://travis-ci.org/tamediadigital/asdf)
[![Circle CI Docs](https://circleci.com/gh/tamediadigital/asdf.svg?style=shield&circle-token=:circle-ci-badge-token)](https://circleci.com/gh/tamediadigital/asdf)
[![Build status](https://ci.appveyor.com/api/projects/status/6o7707y5bk4vxn29?svg=true)](https://ci.appveyor.com/project/yannick/asdf)

# A Simple Document Format

ASDF is a cache oriented string based JSON representation.
Besides, it is a convenient Json Library for D that gets out of your way.
ASDF is specially geared towards transforming high volumes of JSON dataframes, either to new 
JSON Objects or to custom data types.

❗️: Currently all ASDF Method names and all UDAs are in DRAFT state, we might want want make them simpler. Please submit an Issue if you have input.

#### Why ASDF?

- ASDF is fast. It can be really helpful if you have gigabytes of JSON line separated values.
- ASDF is simple. It uses D's modelling power to make you write less boilerplate code.
- ASDF is tested and used in production for real World JSON generated by millions of web clients (we call it _the great fuzzer_).

see also [github.com/tamediadigital/je](https://github.com/tamediadigital/je) a tool for fast extraction of json properties into a csv/tsv.

#### Simple Example

1. define your struct
2. call `serializeToJson` ( or `serializeToJsonPretty` for pretty printing! )
3. profit! 

```D
import asdf;

struct Simple
{
	string name;
	ulong level;
}

void main()
{
	auto o = Simple("asdf", 42);
	string data = `{"name":"asdf","level":42}`;
	assert(o.serializeToJson() == data);
	assert(data.deserialize!Simple == o);
}
```
#### Documentation

See ASDF [API](http://docs.asdf.dlang.io) and [Specification](https://github.com/tamediadigital/asdf/blob/master/SPECIFICATION.md).

#### I/O Speed

 - Reading JSON line separated values and parsing them to ASDF - 300+ MB per second (SSD).
 - Writing ASDF range to JSON line separated values - 300+ MB per second (SSD).

#### Fast setup with the dub package manager

[![Dub version](https://img.shields.io/dub/v/asdf.svg)](http://code.dlang.org/packages/asdf)

[Dub](https://code.dlang.org/getting_started) is the D's package manager.
You can create a new project with:

```
dub init <project-name>
```

Now you need to edit the `dub.json` add `asdf` as dependency and set its targetType to `executable`.

```json
{
	...
	"dependencies": {
		"asdf": "~><current-version>"
	},
	"targetType": "executable",
	"dflags-ldc": ["-mcpu=native"]
}
```

Now you can create a main file in the `source` and run your code with 
```
dub
```
Flags `--build=release` and `--compiler=ldmd2` can be added for a performance boost:
```
dub --build=release --compiler=ldmd2
```

`ldmd2` is a shell on top of [LDC (LLVM D Compiler)](https://github.com/ldc-developers/ldc).
`"dflags-ldc": ["-mcpu=native"]` allows LDC to optimize ASDF for your CPU.

Instead of using `-mcpu=native`, you may specify additional instruction set for a target with `-mattr`.
For example, `-mattr=+sse4.2`. ASDF has specialized code for
[SSE4.2](https://en.wikipedia.org/wiki/SSE4#SSE4.2 instruction set).

#### Compatibility

- LDC (LLVM D Compiler) >= `1.1.0-beta2` (recommended compiler).
- DMD (reference D compiler) >= `2.072.1`.

#### Main transformation functions

| uda | function |
| ------------- |:-------------:|
| `@serializationKeys("bar_common", "bar")` | tries to read the data from either property. saves it to the first one |
| `@serializationKeysIn("a", "b")` | tries to read the data from `a`, then `b`. last one occuring in the json wins |
| `@serializationKeyOut("a")` | writes it to `a` |
| `@serializationMultiKeysIn(["a", "b", "c"])`  | tries to get the data from a sub object. this has not optimal performance yet if you are using more than 1 serializationMultiKeysIn in an object |
| `@serializationIgnore` | ignore this property completely |
| `@serializationIgnoreIn` | don't read this property |
| `@serializationIgnoreOut` | don't write this property |
| `@serializationScoped` | Dangerous! non allocating strings. this means data can vanish if the underlying buffer is removed.  |
| `@serializedAs!string` | call to!string |
| `@serializationTransformIn!fin` | call function `fin` to transform the data |
| `@serializationTransformOut!fout`  | run function `fout` on serialization, different notation |
| `@serializationFlexible`  | be flexible on the datatype on reading, e.g. read long's that are wrapped as strings |


please also look into the Docs or Unittest for concrete examples!

#### ASDF Example (incomplete)

```D
import std.algorithm;
import std.stdio;
import asdf;

void main()
{
	auto target = Asdf("red");
	File("input.jsonl")
		// Use at least 4096 bytes for real wolrd apps
		.byChunk(4096)
		// 32 is minimal value for internal buffer. Buffer can be realocated to get more memory.
		.parseJsonByLine(4096)
		.filter!(object => object
			// opIndex accepts array of keys: {"key0": {"key1": { ... {"keyN-1": <value>}... }}}
			["colors"]
			// iterates over an array
			.byElement
			// Comparison with ASDF is little bit faster
			//   then compression with a string.
			.canFind(target))
			//.canFind("red"))
		// Formatting uses internal buffer to reduce system delegate and system function calls
		.each!writeln;
}
```

##### Input

Single object per line: 4th and 5th lines are broken.

```json
null
{"colors": ["red"]}
{"a":"b", "colors": [4, "red", "string"]}
{"colors":["red"],
	"comment" : "this is broken (multiline) object"}
{"colors": "green"}
{"colors": "red"]}}
[]
```

##### Output

```json
{"colors":["red"]}
{"a":"b","colors":[4,"red","string"]}
```


#### JSON and ASDF Serialization Examples

##### Simple struct or object
```d
struct S
{
	string a;
	long b;
	private int c; // private feilds are ignored
	package int d; // package feilds are ignored
	// all other fields in JSON are ignored
}
```

##### Selection
```d
struct S
{
	// ignored
	@serializationIgnore int temp;
	
	// can be formatted to json
	@serializationIgnoreIn int a;
	
	//can be parsed from json
	@serializationIgnoreOut int b;
}
```

##### Key overriding
```d
struct S
{
	// key is overrided to "aaa"
	@serializationKeys("aaa") int a;

	// overloads multiple keys for parsing
	@serializationKeysIn("b", "_b")
	// overloads key for generation
	@serializationKeyOut("_b_")
	int b;
}
```

##### User-Defined Serialization
```d
struct DateTimeProxy
{
	DateTime datetime;
	alias datetime this;

	static DateTimeProxy deserialize(Asdf data)
	{
		string val;
		deserializeScopedString(data, val);
		return DateTimeProxy(DateTime.fromISOString(val));
	}

	void serialize(S)(ref S serializer)
	{
		serializer.putValue(datetime.toISOString);
	}
}
```

```
//serialize a Doubly Linked list into an Array
struct SomeDoublyLinkedList
{
	@serializationIgnore DList!(SomeArr[]) myDll;
	alias myDll this;

	//no template but a function this time!
	void serialize(ref AsdfSerializer serializer)
    {
        auto state = serializer.arrayBegin();
        foreach (ref elem; myDll)
        {
            serializer.elemBegin;
            serializer.serializeValue(elem);
        }
        serializer.arrayEnd(state);
    }   
}
```

##### Serialization Proxy
```d
struct S
{
	@serializedAs!DateTimeProxy DateTime time;
}
```


##### Finalizer
If you need to do additional calculations or etl transformations that happen to depend on the deserialized data use the `finalizeDeserialization` method.

```d
struct S
{
	string a;
	int b;

	@serializationIgnoreIn double sum;

	void finalizeDeserialization(Asdf data)
	{
		auto r = data["c", "d"];
		auto a = r["e"].get(0.0);
		auto b = r["g"].get(0.0);
		sum = a + b;
	}
}
assert(`{"a":"bar","b":3,"c":{"d":{"e":6,"g":7}}}`.deserialize!S == S("bar", 3, 13));
```

##### serializationFlexible
```D
static struct S
{
	@serializationFlexible uint a;
}

assert(`{"a":"100"}`.deserialize!S.a == 100);
assert(`{"a":true}`.deserialize!S.a == 1);
assert(`{"a":null}`.deserialize!S.a == 0);
 ```

