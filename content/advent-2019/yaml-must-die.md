+++
author = ["Gary Miller"]
title = "Yaml Must Die"
linktitle = "Yaml Must Die"
date = "2019-12-18T00:00:00+00:00"
series = ["Type Respecting Object Notation 2019"]
+++

At the recent [GopherConAU](https://gophercon.com.au/) I did a lightning talk poking fun at configuration languages.
There was a great response to the call to kill JSON and YAML.
With the light in my eyes and the excitement of being on stage in front of 300+ people it sounded to me at the time that about two hundred of them were applauding.
Like the democrats a two thirds majority is a pipe dream and the real question is what are the viable alternatives.
<!-- It retrospect and watching the video it was probably more like 20. -->


Yaml isn’t going anywhere in a hurry, it is just too easy and seductive to conjure up something and defer validation and semantics to later.
But the piper always comes calling and the price needs to be paid.
Many of us have had experience with enterprise systems with multiple interpretations of complex data.
It is not uncommon for the only spec to be the sources code and the documentation other projects' config files.
<!-- If you've done any Kubernetes recently this probably sounds familiar.  -->

Lets call the current situations *Dynamic Config* in comparison to a possible future of  *Meta Schema Config*.
This is technically achievable, but impractical and useable with Json-metaschema (or potentially Yaml schemas, schemaing Yaml schemas).


There are real alternatives, the key is to have a *meta protocol* in the schema language.
The trick with Meta Schemas is to have two syntaxes one for the schema language and a separate one for the value (or annotation language).
In protocol buffers the meta protocol is the `extend` and `option` keyword and the bracket based option operators for fields.
The schema language uses the `message`, `oneof` and primative keys words, and the value syntax is basically the same as `json-6`.

In this blog post we are going to look into using protocol buffers for capturing the content and semantics of configuration.

## Worked example

<!-- This is going to get a bit meta. -->

The command line arguments for the protoc go plugin (protoc-gen-go) is a simple example of dynamic config.
To get a handle on the command line arguments (eg. `paths`) one really needs to go to the [source](https://github.com/golang/protobuf/blob/v1.3.2/protoc-gen-go/generator/generator.go#L466).
Others have created alternatives to the protoc tool chain configured in, guess what, Yaml (eg [buf.build](https://buf.build/docs/build-configuration)).

So, the worked example is to capture the protoc and go plugin config in protobuf.
Creating a Golang CLI that reads a config in the protobuf text format and outputs the protoc command line to execute.

Lets start by understanding the `protoc` command line we are going to use.

```
protoc --go_out=paths=source_relative:. pb/proto-cla.proto
```

The `--NAME_out` tells `protoc` to lookup for an executable name `protoc-gen-NAME` to use as a plugin generator`*`.
The text between the `=` and the `:` are passed to the plugin.
The dot after the colon is the root of the output directory for the generated files.
The `paths=source_relative` tells the plugin to ignore the `option go_package` when constructing the file of the output files.

Our starting proto file looks like

```
syntax = "proto3";

package gopheracademy;
option go_package = "github.com/wxio/protoc_cla/pb";
```
Note: if you don't have the `option go_package` then you don't need the `paths` and the invocation looks like `protoc --go_out=. pb/proto-cla.proto`. The downside is you get to a point where you can't import protos and still generate valid go code. Technically you don't need the `package` either.
We are sticking the packages so as not to fall into this future hole.


Lets read some of the proto-gen-go source to see what config we want to capture.

```go
type pathType int

const (
	pathTypeImport pathType = iota
	pathTypeSourceRelative
)

func (g *Generator) CommandLineParameters(parameter string) {
        ...
		case "paths":
			switch v {
			case "import":
				g.pathType = pathTypeImport
			case "source_relative":
				g.pathType = pathTypeSourceRelative
			default:
				g.Fail(fmt.Sprintf(`Unknown path type %q: want "import" or "source_relative".`, v))
            }
        ...
}
```

becomes

```go
syntax = "proto3";

package gopheracademy;
option go_package = "github.com/wxio/protoc_cla/pb";

message ProtocCLA {
    Generator gen = 1;
    repeated string file = 2;
}

message Generator {
    string output_dir = 1;
    oneof plugin {
        Go go = 2;
    }
}

message Go {
    Paths paths = 1;
}

enum Paths {
    import = 0;
    source_relative = 1;
}
```

Lets generate the go for this protobuf and write a main, creating an instance and print it.
The generated code is the nices API, specifically when creating `oneof`s.
The point of this example is to output the proto text format and use that as the config format.
`Oneof`s are more usable in this format.

```
protoc --go_out=paths=source_relative:. pb/protoc-cla.proto
```

```
package main

import (
	"os"

	"github.com/golang/protobuf/proto"
	"github.com/wxio/protoc_cla/pb"
)

func main() {
	protoCLA := pb.ProtocCLA{
		File: []string{"pb/protoc-cla.proto"},
		Gen: []*pb.Generator{
			&pb.Generator{
				OutputDir: ".",
				Plugin: &pb.Generator_Go{
					Go: &pb.Go{
						Paths: pb.Paths_source_relative,
					},
				},
			},
		},
	}
	proto.MarshalText(os.Stdout, &protoCLA)
}
```

```
$ go run main.go  | tee proto.cfg
gen: <
  output_dir: "."
  go: <
    paths: source_relative
  >
>
file: "pb/proto-cla.proto"
```

Now we implement a go program to consume the config and output the protoc command line.

```
func (*convert) Run() error {
	cfg, err := ioutil.ReadAll(os.Stdin)
	if err != nil {
		return err
	}
	protoCLA := pb.ProtocCLA{}
	if err = proto.UnmarshalText(string(cfg), &protoCLA); err != nil {
		return err
	}
	sb := strings.Builder{}
	sb.WriteString("protoc ")
	switch plug := protoCLA.Gen.Plugin.(type) {
	case *pb.Generator_Go:
		sb.WriteString("--go_out=paths=")
		sb.WriteString(pb.Paths_name[int32(plug.Go.Paths)])
	default:
		return fmt.Errorf("unknown plugin %T", plug)
	}
	sb.WriteString(":")
	sb.WriteString(protoCLA.Gen.OutputDir)
	sb.WriteString(" ")
	for _, file := range protoCLA.File {
		sb.WriteString(file)
		sb.WriteString(" ")
	}
	fmt.Printf("%s\n", sb.String())
	return nil
}
```

```
cat proto.cfg | go run main.go convert
cat proto.cfg | go run main.go convert | sh
```

## Next steps

Ok, so we have a schema for our config.
You might say we haven't achieve very much and I would agree.
The next step starts to become more interesting.
Here we schema our schema, the promised meta schema.

This is a cut down and slightly modified version of [envoyproxy/proto-gen-validate](https://github.com/envoyproxy/protoc-gen-validate/blob/v0.1.0/validate/validate.proto).

```
syntax = "proto3";

package gopheracademy;
option go_package = "github.com/wxio/protoc_cla/pb";

import "google/protobuf/descriptor.proto";

// Validation rules applied at the field level
extend google.protobuf.FieldOptions {
    // Rules specify the validations to be performed on this field. By default,
    // no validation is performed against a field.
    repeated FieldRules rules = 1072;
}

message FieldRules {
    oneof type {
        StringRules   string   = 14;
        MessageRules  message  = 17;
        RepeatedRules repeated = 18;
    }
}

// StringRules describe the constraints applied to `string` values
message StringRules {
    // MinLen specifies that this field must be the specified number of
    // characters (Unicode code points) at a minimum. Note that the number of
    // characters may differ from the number of bytes in the string.
    uint64 min_len = 2;
}

...
```

The protoc-cla.proto is now decorated with validation rules.

```
syntax = "proto3";

package gopheracademy;
option go_package = "github.com/wxio/protoc_cla/pb";

import "pb/validation.proto";

message ProtocCLA {
    Generator gen = 1
        [(rules) = { message : { required : true } }]
    ;
    repeated string file = 2
        [(rules) = {repeated : { min_items : 1, unique : true} }]
    ;
}

message Generator {
    string output_dir = 1  [(rules) = { string : { min_len : 1 } }];
    oneof plugin {
        Go go = 2;
    }
}

message Go {
    Paths paths = 1;
}

enum Paths {
    import = 0;
    source_relative = 1;
}
```

And a small Go program using proto reflect shows we can access the meta information at runtime.

```
func (*validators) Run() {
	dumpValidators("", &pb.ProtocCLA{})
}

func dumpValidators(path string, msg descriptor.Message) {
	_, md := descriptor.ForMessage(msg)
	for _, fi := range md.Field {
		fi.GetOptions()
		rule, err := proto.GetExtension(fi.GetOptions(), pb.E_Rules)
		if rule != nil && err == nil {
			fmt.Printf("%s%s : %v\n", path, *fi.Name, rule)
		}
	}
	for _, fi := range md.Field {
		if *fi.Type == protobuf.FieldDescriptorProto_TYPE_MESSAGE {
			// TODO handle slices aka repeated messages
			msg2Typ := proto.MessageType(fi.GetTypeName()[1:])
			msg2 := reflect.New(msg2Typ.Elem()).Interface()
			dumpValidators(path+*fi.Name+":", msg2.(descriptor.Message))
		}
	}
}
```

```
$ go run main.go validators
gen : [message:<required:true > ]
file : [repeated:<min_items:1 unique:true > ]
gen:output_dir : [string:<min_len:1 > ]
```

We could easily go deeper and schema the meta schema.
An example in this case is a schema to capture the documentation in the validation.proto.

Or could go broader specifying generators for the swagger and grpc-gateway generators. 
In this case a meta schema capturing some invarient when using these two generator together could be specified.



### Notes
`*` there are builting generators, which technically aren't plugins. See `protoc -h`