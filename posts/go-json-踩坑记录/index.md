# Go json 踩坑记录


JSON 是一种轻量级的数据交换格式，语言易于阅读和编写。它用于传输由属性值或序列化值组成的数据对象，已经被广泛应用。Go 通过 `encoding/json` 库提供了完整的 JSON 数据序列化和反序列化支持。然而，使用这种技术时需要注意一些关键点。

Go 可使用 `json.Marshal()` 便捷的获取 JSON 数据，查看该函数对应的 godoc 文档，里面有这么一段话:
```
String values encode as JSON strings coerced to valid UTF-8, replacing invalid bytes with the Unicode replacement rune. So that the JSON will be safe to embed inside HTML &lt;script&gt; tags, the string is encoded using HTMLEscape, which replaces &#34;&lt;&#34;, &#34;&gt;&#34;, &#34;&amp;&#34;, U&#43;2028, and U&#43;2029 are escaped to &#34;\u003c&#34;,&#34;\u003e&#34;, &#34;\u0026&#34;, &#34;\u2028&#34;, and &#34;\u2029&#34;. This replacement can be disabled when using an Encoder, by calling SetEscapeHTML(false).
```

`json.Marshal()` 在进行序列化时，会进行 HTMLEscape 编码，会将 &#34;&lt;&#34;, &#34;&gt;&#34;, &#34;&amp;&#34;, U&#43;2028, 及 U&#43;2029 转码成 &#34;\u003c&#34;,&#34;\u003e&#34;, &#34;\u0026&#34;, &#34;\u2028&#34;, 和 &#34;\u2029&#34;。这在正常使用时是没有问题的，但如果在对接第三方需要对 JSON 字符串进行取摘要比对时，如果一方未进行 HTMLEscape 编码，取出的摘要便会天差地别。上述文档中也给出了解决方法，通过 `SetEscapeHTML(false)` 来禁用便可。方法如下：

```go
bf := bytes.NewBuffer([]byte{})
jsonEncoder := json.NewEncoder(bf)
jsonEncoder.SetEscapeHTML(false)
_ = jsonEncoder.Encode(body)

jsonStr := bf.String()
```

但是这样使用仍然会有些问题，`json.Encoder.Encode()` 会在 JSON 字符串后接一个 **换行符**，该函数源码如下：
```go
// Encode writes the JSON encoding of v to the stream,
// followed by a newline character.
//
// See the documentation for Marshal for details about the
// conversion of Go values to JSON.
func (enc *Encoder) Encode(v interface{}) error {
	if enc.err != nil {
		return enc.err
	}
	e := newEncodeState()
	err := e.marshal(v, encOpts{escapeHTML: enc.escapeHTML})
	if err != nil {
		return err
	}

	// Terminate each value with a newline.
	// This makes the output look a little nicer
	// when debugging, and some kind of space
	// is required if the encoded value was a number,
	// so that the reader knows there aren&#39;t more
	// digits coming.
	e.WriteByte(&#39;\n&#39;)

	b := e.Bytes()
	if enc.indentPrefix != &#34;&#34; || enc.indentValue != &#34;&#34; {
		if enc.indentBuf == nil {
			enc.indentBuf = new(bytes.Buffer)
		}
		enc.indentBuf.Reset()
		err = Indent(enc.indentBuf, b, enc.indentPrefix, enc.indentValue)
		if err != nil {
			return err
		}
		b = enc.indentBuf.Bytes()
	}
	if _, err = enc.w.Write(b); err != nil {
		enc.err = err
	}
	encodeStatePool.Put(e)
	return err
}
```

可以看出，该函数在进行序列化后又写入了一个 `\n` 字符。如果不需要该字符，则需要额外的将其剔除：
```
jsonStr := string(bf.Bytes()[:bf.bf.Len()])
```


---

> 作者: chirizcc  
> URL: http://localhost:1313/posts/go-json-%E8%B8%A9%E5%9D%91%E8%AE%B0%E5%BD%95/  

