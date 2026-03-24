# 08 — Migrate MessagePack Serialization (`msgpack.fs`)

**Status:** pending

## Overview

Translate `msgpack.fs` to `MsgPack.cs`. This file defines a custom `IMessagePackFormatter<obj>` and resolver for the `MessagePack-CSharp` library. The formatter is critical for the Neovim RPC protocol — it serializes/deserializes the `obj` type used throughout the RPC layer. The `MessagePack-CSharp` library is a C# library used directly from F#, so this translation is mostly mechanical.

## Motivation

The `MessagePack-CSharp` library (`MessagePack` NuGet) is already a C# library. The F# code is thin glue code that implements `IMessagePackFormatter<'T>` (where `'T` is `obj` / `System.Object`). The translation is direct — the interface is unchanged in C#.

## Steps

1. **Create `MsgPack.cs`** with the formatter:

   ```csharp
   using MessagePack;
   using MessagePack.Formatters;
   using MessagePack.Resolvers;

   public class MsgPackObjectFormatter : IMessagePackFormatter<object>
   {
       public static readonly MsgPackObjectFormatter Instance = new();

       public void Serialize(ref MessagePackWriter writer, object value, MessagePackSerializerOptions options)
       {
           // Serialize based on runtime type
           switch (value) {
               case null:           writer.WriteNil(); break;
               case bool b:         writer.Write(b); break;
               case int i:          writer.Write(i); break;
               case long l:         writer.Write(l); break;
               case uint u:         writer.Write(u); break;
               case ulong ul:       writer.Write(ul); break;
               case float f:        writer.Write(f); break;
               case double d:       writer.Write(d); break;
               case string s:       writer.Write(s); break;
               case byte[] bytes:   writer.Write(bytes); break;
               case object[] arr: {
                   writer.WriteArrayHeader(arr.Length);
                   foreach (var item in arr)
                       Serialize(ref writer, item, options);
                   break;
               }
               case Dictionary<object, object> dict: {
                   writer.WriteMapHeader(dict.Count);
                   foreach (var kv in dict) {
                       Serialize(ref writer, kv.Key, options);
                       Serialize(ref writer, kv.Value, options);
                   }
                   break;
               }
               default: throw new NotSupportedException($"Unsupported type: {value.GetType()}");
           }
       }

       public object Deserialize(ref MessagePackReader reader, MessagePackSerializerOptions options)
       {
           switch (reader.NextMessagePackType) {
               case MessagePackType.Nil:     reader.ReadNil(); return null!;
               case MessagePackType.Boolean: return reader.ReadBoolean();
               case MessagePackType.Integer: return reader.ReadInt64();  // unify to long
               case MessagePackType.Float:   return reader.ReadDouble();
               case MessagePackType.String:  return reader.ReadString()!;
               case MessagePackType.Binary:  return reader.ReadBytes()!.ToArray();
               case MessagePackType.Array: {
                   int len = reader.ReadArrayHeader();
                   var arr = new object[len];
                   for (int i = 0; i < len; i++)
                       arr[i] = Deserialize(ref reader, options);
                   return arr;
               }
               case MessagePackType.Map: {
                   int len = reader.ReadMapHeader();
                   var dict = new Dictionary<object, object>(len);
                   for (int i = 0; i < len; i++) {
                       var k = Deserialize(ref reader, options);
                       var v = Deserialize(ref reader, options);
                       dict[k] = v;
                   }
                   return dict;
               }
               default: throw new NotSupportedException($"Unsupported msgpack type: {reader.NextMessagePackType}");
           }
       }
   }
   ```

2. **Create the resolver**:
   ```csharp
   public class MsgPackResolver : IFormatterResolver
   {
       public static readonly MsgPackResolver Instance = new();

       public IMessagePackFormatter<T>? GetFormatter<T>() {
           if (typeof(T) == typeof(object))
               return (IMessagePackFormatter<T>)(object)MsgPackObjectFormatter.Instance;
           return StandardResolver.Instance.GetFormatter<T>();
       }
   }
   ```

3. **Create shared `MessagePackSerializerOptions`**:
   ```csharp
   public static class MsgPackOptions {
       public static readonly MessagePackSerializerOptions Default =
           MessagePackSerializerOptions.Standard.WithResolver(MsgPackResolver.Instance);
   }
   ```

4. **Check the original `msgpack.fs`** for the exact integer type mapping. In the F# version, the Neovim protocol sends integers as `int32` — check whether the deserializer should return `int32` or `int64` for `MessagePackType.Integer`. Neovim uses 64-bit integers in its protocol; map to `long`.

5. **Verify that `Dictionary<object, object>`** is the correct type for msgpack maps — the F# code uses `hashmap<obj,obj>` which is `Dictionary<obj,obj>`. Confirm the type is consistent with how `def.fs` active patterns use `hashmap<obj,obj>`.

6. **Update all call sites** in `neovim.cs` (the RPC reader loop) to use `MsgPackOptions.Default`.

## References

- Source: `msgpack.fs`
- MessagePack-CSharp docs: https://github.com/MessagePack-CSharp/MessagePack-CSharp
- `IMessagePackFormatter<T>`: interface in `MessagePack.Formatters`
