# How to parse a MIME email in Golang

## Introduction
I wanted to learn how to parse a MIME email using Go. As an exercice, I decided to write a simple tool that would read a MIME email from `stdio` and write all its MIME parts into separated files, based on the type of each part. Then I wanted to consolidate my learnings into this simple paper. 

## Separating the header and the body of the message
First thing first, the header and the body of the message have to be separated from the email. Golang provides the `net/mail` package and its `mail.ReadMessage()` function for this. `ReadMessage()` returns a `Message` structure which has the two fields we are looking for: a Header and a Body. 

```golang
m, err := mail.ReadMessage(os.Stdin)
```

The `Body` is just an `io.Reader()` which can be used to read the content of the message, while the `Header` field is a map of slices of strings (`type Header map[string][]string`) which holds the message header values. The keys of the maps are the names of the header fields, so, for example, the subject of the message is stored in `m.Header["Subject"][0]`.

```golang
fmt.Println("Subject:", m.Header["Subject"][0])
```

Why a slice of strings for each of the header entry ? Because the same header field can be found several times in the header of the message. For example, there is a new `Received` value for each of the mail relays the message went trough. Because of this, it is recommended to use to `Header.Get()` function that returns only the first value, in a case insensitive way, and simply returns `""` if there is no value associated with the key or the key does not exist in the map. 

## Understanding the content type of the message
For a MIME email, the most important field is `Content-Type`:

```golang
fmt.Println("Content-Type:", m.Header.Get("Content-Type"))
```

```
Content-Type: multipart/alternative; boundary=bcaec520ea5d6918e204a8cea3b4
```

Be warned that the `Content-Type` entry is not always present in all messages. Only the `Date:`, `From:` and either `To:` or `Bcc:` are mandatory. However, it is required for a MIME email.

When present, the `Content-Type` is made of several values separated by semi-colon. The first one (`multipart/alternative` in this example) indicates the format used for the body of the message. The next ones are key/value pairs that give information on how to decode the body. For example `boundary` indicates the string used to separate the various MIME parts inside the body.

The most common MIME formats for a message body are:

- `text/plain`: the message body is simple text,  
- `text/html`: for an HTML message body, 
- `multipart/mixed`: for a message body with contents of various formats, including attached files, each content being given as a separate MIME part in the message body,
- `multipart/alternative`: when the same content is provided into different formats, most of the time text and HTML, to allow display on various devices, each MIME part holds one the provided format.

## A MIME email example
Below is an example of a very basic MIME email message, where the same content (`multipart/alternative`) is provided in different formats (`text/plain` and `text/html`) through two different MIME parts, each MIME part being separated by the `bcaec520ea5d6918e204a8cea3b4` boundary (example taken from [Send Html page As Email using "mutt"](https://stackoverflow.com/questions/6805783/send-html-page-as-email-using-mutt)).

```
Subject: test html mail
From: sender@example.com
To: recipient@example.com
Content-Type: multipart/alternative; boundary=bcaec520ea5d6918e204a8cea3b4

--bcaec520ea5d6918e204a8cea3b4
Content-Type: text/plain; charset=ISO-8859-1

*hi!*

--bcaec520ea5d6918e204a8cea3b4
Content-Type: text/html; charset=ISO-8859-1
Content-Transfer-Encoding: quoted-printable

<p><b>hi!</b></p>

--bcaec520ea5d6918e204a8cea3b4--
```

Most of the email solutions allow to retrieve emails in raw format that can then be parsed using this `parseMIMEmail` tool.

## Parsing the `Content-Type` information string

Once the `Content-Type` string has been retrieved with `m.Header.Get("Content-Type")`, it is possible to parse it using the function `ParseMediaType()` from the `mime` package. This function returns
- `mediatype`: a string, converted to lower case, that holds the media type as discussed above.
- `params`: a map of string, with a value for each key-value pairs extracted from the string. So, if a `boundary` parameter is present in the `Content-Type`, `params["boundary"]`holds its value. The key is converted to lower case, but not the associated value.
- `err`: in case something when wrong during the parsing

In my case, I used the following code :

```golang
	mediaType, params, err := mime.ParseMediaType(m.Header.Get("Content-Type"))
	if err != nil {
		log.Fatal(err)
	}

	if !strings.HasPrefix(mediaType, "multipart/") {
		log.Fatalf("Not a multipart MIME message\n")
	}
```

## Parsing the body of the message and all its MIME parts

Two important things have to be understood there :

1. A dedicated Reader to read the MIME format is needed. The `mime/multipart` package and its `mime.NewReader()` function provide that to us.
2. MIME parts can be (and usually are) nested : a multipart MIME part can itself contains other multipart MIME parts, which means a recursive reading of the MIME parts is required.

MIME parts are separated by boundaries. The boundary used is made from the boundary value retrieved previously from the `Content-Type` and parsed using the `mime.ParseMediaType()` function. MIME parts of the same level share the same boundaries. To instantiate a new MIME reader, `mime.NewReader()` must be given 2 arguments: the reader to which the data from the MIME part can be retrieved and the boundary used to separate the MIME parts :

```golang
// Separate header and body of the message read from stdin
m, err := mail.ReadMessage(os.Stdin)

// Retrieve the Content-Type header and parse it to get the boundary value
mediaType, params, err := mime.ParseMediaType(m.Header.Get("Content-Type"))

// Instantiate a new MIME reader from the body of the message using the boundary
reader := multipart.NewReader(m.Body, params["boundary"])
```

And because a MIME part can itself be a new multipart MIME part, is better to have a dedicated ParsePart() function that can be called recursively:

```golang

func main() {

	m, err := mail.ReadMessage(os.Stdin)
	mediaType, params, err := mime.ParseMediaType(m.Header.Get("Content-Type"))
	ParsePart(m.Body, params["boundary"])
	
}

func ParsePart(mime_data io.Reader, boundary string) {

	// Instantiate a new io.Reader dedicated to MIME multipart parsing
	// using multipart.NewReader()
	reader := multipart.NewReader(mime_data, boundary)
	if reader == nil {
		return
	}

	// Go through each of the MIME part of the message Body with NextPart(),
	for {

		new_part, err := reader.NextPart()
		if err == io.EOF {
			break
		}
		if err != nil {
			fmt.Println("Error going through the MIME parts -", err)
			break
		}

		mediaType, params, err := mime.ParseMediaType(new_part.Header.Get("Content-Type"))
		
		if err == nil && strings.HasPrefix(mediaType, "multipart/") {
		
			// This is a new multipart to be handled recursively
			ParsePart(new_part, params["boundary"])

		} else {

			// Not a new nested multipart.
			// We can do something here with the data of this single MIME part.

		}

	}

}
```

## Decoding a MIME part

The data of a MIME part can be read using `ioutil.ReadAll()`. 

```golang
new_part, err := reader.NextPart()
part_data, err := ioutil.ReadAll(part)
```

MIME part data is encoded, and the encoding information can be retrieved from the `Content-Transfer-Encoding` header of **this** MIME part (not to confused with the header of the message).

```golang
content_transfer_encoding := part.Header.Get("Content-Transfer-Encoding")
```

See [https://www.w3.org/Protocols/rfc1341/5_Content-Transfer-Encoding.html](https://www.w3.org/Protocols/rfc1341/5_Content-Transfer-Encoding.html) for a description of the `Content-Transfer-Encoding` header.

Possible encoding types are

```
Content-Transfer-Encoding := "BASE64" / "QUOTED-PRINTABLE" / 
                             "8BIT"   / "7BIT" / 
                             "BINARY" / x-token
```
 
8BIT, 7BIT and BINARY imply no encoding were performed. x-token are 'X-' values used to indicate non standard encoding. BASE64 and QUOTED-PRINTABLE can be decoded using the `encoding/base64` and `mime/quotedprintable`packages:

```golang
part_data, err := ioutil.ReadAll(part)
content_transfer_encoding := part.Header.Get("Content-Transfer-Encoding")

switch {

		case strings.Compare(content_transfer_encoding, "BASE64") == 0:
			decoded_content, err := base64.StdEncoding.DecodeString(string(part_data))
			if err != nil {
				log.Println("Error decoding base64 -", err)
			} else {
				// do something with the decoded content
			}

		case strings.Compare(content_transfer_encoding, "QUOTED-PRINTABLE") == 0:
			decoded_content, err := ioutil.ReadAll(quotedprintable.NewReader(bytes.NewReader(part_data)))
			if err != nil {
				log.Println("Error decoding quoted-printable -", err)
			} else {
				// do something with the decoded content
			}

		default:
			// Data is not encoded, do something with part_data

	}
```


## Conclusion: step-by-step summary

1) Parse the message to separate the Header and the Body with `mail.ReadMessage()`

```golang
	m, err := mail.ReadMessage(os.Stdin)
	if err != nil {
		log.Fatalln("Parse mail KO -", err)
	}
```

2) Retrieve the `Content-Type` information string from the header to get the MIME structure of the message with `Header.Get()`, and parse the `Content-Type` information string with `mime.ParseMediaType()`

```golang
	mediaType, params, err := mime.ParseMediaType(m.Header.Get("Content-Type"))
	if err != nil {
		log.Fatal(err)
	}

	if !strings.HasPrefix(mediaType, "multipart/") {
		log.Fatalf("Not a multipart MIME message\n")
	}
```

3) Instantiate a `new io.Reader` dedicated to MIME multipart parsing with `multipart.NewReader()`

```golang
	reader := multipart.NewReader(m.Body, params["boundary"])
	if reader == nil {
		return
	}
```

4) Go through each of the MIME part of the message Body with `NextPart()`

```golang
	for {

		new_part, err := reader.NextPart()
		if err == io.EOF {
			break
		}
		if err != nil {
			fmt.Println("Error going through the MIME parts -", err)
			break
		}

		// Do something with the new_part being processed.
		// new_part can itself be a nested new multipart part,
		// requiring some kind of recursive processing.

	}
``` 
 
6) Read the content of the MIME part with `ioutil.ReadAll()` and decode it depending of its `Content-Transfer-Encoding` information

```golang
	part_data, err := ioutil.ReadAll(new_part)
	if err != nil {
		log.Println("Error reading MIME part data -", err)
		return
	}

	content_transfer_encoding := strings.ToUpper(new_part.Header.Get("Content-Transfer-Encoding"))

	switch {

		case strings.Compare(content_transfer_encoding, "BASE64") == 0:
			decoded_content, err := base64.StdEncoding.DecodeString(string(part_data))
			if err != nil {
				log.Println("Error decoding base64 -", err)
			} else {
				ioutil.WriteFile(filename, decoded_content, 0644)
			}

		case strings.Compare(content_transfer_encoding, "QUOTED-PRINTABLE") == 0:
			decoded_content, err := ioutil.ReadAll(quotedprintable.NewReader(bytes.NewReader(part_data)))
			if err != nil {
				log.Println("Error decoding quoted-printable -", err)
			} else {
				ioutil.WriteFile(filename, decoded_content, 0644)
			}

		default:
			ioutil.WriteFile(filename, part_data, 0644)

	}	
```

##Bonus - reading RFC 2047 headers
Despite being limited to 7 bits ASCII characters, it is possible to have special characters in the headers of a message thanks to RFC 2047. For example, to allow a message subject like "Subject: ¡Hola, señor!", the Subject field of the message header has to be encoded using RFC 2047 like this: `Subject: =?iso-8859-1?Q?=A1Hola,_se=F1or!?=`. The `Subject`, `To` and `From` fields of the header are the most often encoded using RFC 2047. 

Fortunately, with the `mime` package and its `WordDecoder.HeaderDecode()` function there is no point in trying to understand how RFC 2047 works. All is needed is to create a new decoder for RFC encoded data with `new(mime.WordDecoder)` then use the `HeaderDecode()` method. It is how I implemented the display of the main header fields of the message:

```golang
	dec := new(mime.WordDecoder)
	from, _ := dec.DecodeHeader(m.Header.Get("From"))
	to, _ := dec.DecodeHeader(m.Header.Get("To"))
	subject, _ := dec.DecodeHeader(m.Header.Get("Subject"))
	fmt.Println("From:", from)
	fmt.Println("To:", to)
	fmt.Println("Date:", m.Header.Get("Date"))
	fmt.Println("Subject:", subject)
	fmt.Println("Content-Type:", m.Header.Get("Content-Type"))
	fmt.Println()
```


