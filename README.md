[![Dub version](https://img.shields.io/dub/v/asdf.svg)](http://code.dlang.org/packages/asdf)
[![License](https://img.shields.io/dub/l/asdf.svg)](http://code.dlang.org/packages/asdf)
[![codecov.io](https://codecov.io/github/tamediadigital/asdf/coverage.svg?branch=master)](https://codecov.io/github/tamediadigital/asdf?branch=master)
[![Build Status](https://travis-ci.org/tamediadigital/asdf.svg?branch=master)](https://travis-ci.org/tamediadigital/asdf)
[![Circle CI Docs](https://circleci.com/gh/tamediadigital/asdf.svg?style=shield&circle-token=:circle-ci-badge-token)](https://circleci.com/gh/tamediadigital/asdf.svg?style=shield&circle-token=:circle-ci-badge-token)

# A Simple Document Format

ASDF is a cache oriented string based JSON representation.
It allows to easily iterate over JSON arrays/objects multiple times without parsing them.
ASDF does not parse numbers (they are represented as strings).
ASDF values can be removed by setting `deleted` bit on.

For line separated JSON values see `parseJsonByLine` function.
This function accepts a range of chunks instead of a range of lines.

Besides is a convinient Json Library for D that gets out of your way.
ASDF is specially geared towards transforming JSON dataframes, either to new 
JSON Objects or to custom data types.

❗️: Currently all ASDF Method names and all UDAs are in DRAFT state, we might want want make them simpler. Please submit an Issue if you have input.

❗️: ASDF is currently only very loosely validating jsons and with certain functions even silently and on purpose ignoring failing Objects (see below). 

#### Why ASDF?

ASDF is fast. It can be really helpful if you have gigabytes of JSON line separated values.

#### Specification

See [ASDF Specification](https://github.com/tamediadigital/asdf/blob/master/SPECIFICATION.md).

#### I/O Speed

 - Reading JSON line separated values and parsing them to ASDF - 300+ MB per second (SSD).
 - Writing ASDF range to JSON line separated values - 300+ MB per second (SSD).

 - 🚀 for extra speed powerups on X86 you can use `"subConfigurations": { "asdf": "native-sse42" }` in your `dub.json`. Watch it fly!

#### current transformation functions

| uda | function |
| ------------- |:-------------:|
| `@serializationKeys("bar_common", "bar")` | tries to read the data 
| `@serializationKeysIn("a", "b")` | tries to read the data from `b`, then `b` |
| `@serializationMultiKeysIn(["a", "b", "c"])`  | tries to get the data from a sub object. this has not optimal performance yet if you are using more than 1 serializationMultiKeysIn in an object |
| `@serializationIgnore` | ignore this property completely |
| `@serializationIgnoreIn` | don't read this property |
| `@serializationIgnoreOut` | don't write this property |
| `@serializationScoped` | Dangerous! non allocating strings. this means data can vanish if the underlying buffer is removed.  |
| `@serializedAs!string` | call to!string |
| `@serializationTransformIn!fin` | call function `fin` to transform the data |
| ```@serializationTransformOut!fout`  | run function on serialization, different notation |

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
			//.canFind("tadmp5800"))
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
	@serializationIgnore
	int temp;
	
	// can be formatted to json
	@serializationIgnoreIn
	int a;
	
	//can be parsed from json
	@serializationIgnoreOut
	int b;
}
```

##### Key overriding
```d
struct S
{
	// key is overrided to "aaa"
	@serializationKeys("aaa")
	int a;

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
	@serializedAs!DateTimeProxy
	DateTime time;
}
```


##### Finalizer
```d
struct S
{
	string a;
	int b;

	@serializationIgnoreIn
	double sum;

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
