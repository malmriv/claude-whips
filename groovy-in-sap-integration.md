---
name: sap-cpi-groovy
description: Author Groovy scripts for the Script step in SAP Cloud Integration (SAP CPI, Integration Suite, Cloud Platform Integration, iFlows). Use this whenever the request involves Groovy inside an iFlow, a processData script, the Camel message with its headers, properties and attachments, XmlSlurper or JsonSlurper over a CPI payload, or any "write me a script for my iFlow" ask. Use it also when the user pastes an existing CPI Groovy script and wants it extended, refactored or fixed. Groovy running inside CPI has non-obvious constraints that generic Groovy knowledge gets wrong on a regular basis, so consult this skill even when the task looks like a trivial snippet.
---

# SAP CPI Groovy authoring

Scripts written for the Script step of an SAP Cloud Integration iFlow. Not plain
Groovy, not Gradle, not Jenkins. The runtime is a Camel exchange, the entry point
is fixed, and several habits from general-purpose Groovy produce code that
compiles locally and fails on the tenant.

Work in this order: ask (section 1), write from the template (section 2), then
check the output against section 10 before returning it.

---

## 1. Ask before writing

A CPI script is a contract with the steps around it. Writing one without knowing
that contract produces plausible code that does not fit the iFlow. Ask all of
the following in a single message, not one at a time:

1. What must the step do, and what makes it a script rather than a palette step?
2. A realistic sample of the incoming payload, and the expected output. One
   example of each is enough; two if the shape varies.
3. Content type in and out: XML, JSON, CSV, plain text, binary.
4. Which values arrive as headers, which as exchange properties, and which the
   script is expected to set for later steps.
5. If any timestamp is involved: UTC, or a specific IANA zone such as
   `Europe/Madrid`? And in which output format?
6. Any charset other than UTF-8, on either side.
7. On missing or malformed data: fail loudly so the exception subprocess picks
   it up, or pass through unchanged?

Never guess a timezone from context. "The user is in Spain" is not a
specification: `Europe/Madrid` is UTC+1 or UTC+2 depending on the date, and the
one thing a hardcoded offset guarantees is a defect twice a year.

Never invent the payload structure. A script written against an imagined
document is worse than no script, because it looks finished.

**When no answer is available**: default to UTC, UTF-8, and fail-loudly, then
state every assumption explicitly in the header comment block of the script.

---

## 2. Canonical template

Copy this shape literally: signature, import block, description block, then the
function with block comments. Everything about the layout below is intentional.

```groovy
import com.sap.gateway.ip.core.customdev.util.Message
import groovy.xml.*

/*
 * ---------------------------------------------------------------------------
 * Purpose : Normalises the customer name in the incoming order document and
 *           exposes the order ID so a later Router step can use it.
 *
 * Input   : Order XML, single <Order> root, no namespaces.
 * Output  : Same document, <Content> rewritten.
 *
 * Reads   : property "defaultCountry"  (set by the preceding Content Modifier)
 * Writes  : property "orderId"
 *           header   "X-Order-Type"    (travels outbound on the next HTTP call)
 *
 * Assumes : payload is well-formed UTF-8 XML.
 * ---------------------------------------------------------------------------
 */

def Message processData(Message message) {

   //Content

   return message
}
```

Rules the template encodes:

| Element | Requirement |
| --- | --- |
| Entry point | `def Message processData(Message message)`, returning `message`. This is the SAP-generated signature; keep it even though `def` plus a return type is redundant Groovy. |
| Message import | `com.sap.gateway.ip.core.customdev.util.Message`, always present. |
| Description block | Between the imports and the function. Purpose, input, output, properties and headers read and written, assumptions. |
| Comments | Block comments on their own lines, marking the phases of the script. Not trailing comments that push lines past 100 characters. |
| Indentation | Four spaces, consistent. The whole body of the function is indented. |
| Auxiliary functions | Below `processData`, each with its own short description comment above it. |

Use a different entry-point name only when the user has explicitly configured
one in the Script step, and say so in the comment block if they have.

---

## 3. The Camel message, operationally

A message is a body plus three side channels. Choosing the wrong channel is one
of the most common design errors in CPI scripts.

| Channel | Lifetime | Leaves the tenant? | Use for |
| --- | --- | --- | --- |
| Body | The payload itself | Yes | The document being transformed |
| Headers | The message | Yes, automatically attached to subsequent HTTP calls | Values the receiver system needs: content type, correlation ID, routing keys |
| Properties | The whole exchange | No | Everything internal: intermediate values, counters, flags, anything sensitive |
| Attachments | The message | Only if the adapter sends them | Optional payload extras, and monitoring attachments via the message log |

Decision rule: **if a later step in this iFlow needs it, use a property. If the
receiving system needs it, use a header.** When in doubt, property.

Never put a token, password, personal data or a full payload in a header. It
propagates outbound without anyone asking, which turns a convenience into a data
leak.

API surface:

```groovy
def body    = message.getBody(String)               // also byte[], java.io.Reader
def propMap = message.getProperties()               // Map, read-only in practice
def hdrMap  = message.getHeaders()

def someProp = message.getProperties().get('name')
def someHdr  = message.getHeaders().get('name')

message.setBody(payloadString)                      // String, byte[] or InputStream
message.setProperty('name', value)
message.setHeader('name', value)
```

Two failure modes worth naming:

**Reading the body twice.** If the body is backed by a stream, the second read
returns empty. Read once into a local variable at the top and work from there.

**Setting a non-serialisable body.** `setBody` should receive a String, a
`byte[]` or an `InputStream`. Handing it a Map, a `GPathResult` or a something else leaves
a Java object in the body that the next step cannot serialise, and the error
appears one step later than the cause.

---

## 4. Non-negotiables

### 4.1 No calls out of the script

No HTTP, no JDBC, no JMS, no SFTP, no mail, no reading a URL. It is technically
possible in Groovy and it is the wrong thing to do: a call made inside a script
is invisible to the iFlow model, unmonitored, untraceable in Message Monitoring,
outside the tenant's connectivity and certificate management, and impossible for
the next person to find. Connectivity belongs to adapters.

If a payload needs enrichment from another system, say so and describe the step
that should do it (Request Reply, Content Enricher, Local Integration Process).

### 4.2 Do not rebuild the middleware

A Groovy script exists for what the palette cannot express. Push back, and name
the step, when a request would reimplement any of these:

| Requested in script | Belongs to |
| --- | --- |
| Routing on a payload value | Router |
| Splitting a collection | Splitter |
| Re-joining split messages | Gather / Join |
| Field-to-field mapping | Message Mapping |
| XML to JSON or JSON to XML | XML to JSON / JSON to XML converter |
| Setting static or simple-expression values | Content Modifier |
| Filtering by XPath | Filter |
| Polling, scheduling, retry | Adapter and iFlow configuration |
| Encryption, signing, PGP | Encryptor / Decryptor / Signer steps |

Writing the script anyway is acceptable when the user knows the trade-off and
asks for it. Writing it silently is not.

### 4.3 Never parse structured data with regular expressions

XML and JSON are not regular languages. A regex works on the sample payload and
breaks on attribute order, self-closing tags, CDATA, namespaces, escaped
entities, unexpected whitespace or a value containing the delimiter. Use a
slurper, always, including for a single field.

The one legitimate use of a regex on a payload is inside a text value that has
already been extracted by a parser.

### 4.4 XML parser class names

Write XML parser imports as `import groovy.xml.*` and instantiate unqualified:
`new XmlSlurper()`, `new XmlParser()`.

Never emit `groovy.util.XmlSlurper` or `groovy.util.XmlParser`, neither as an
import nor as a fully qualified name. Those classes were removed in later Groovy
versions and code written that way fails on most runtimes.

The star import is deliberate. An explicit single-class import of
`groovy.xml.XmlSlurper` fails on older runtimes where that class lives elsewhere,
whereas the star form resolves correctly everywhere. Do not "tidy" it into a
specific import.

For everything else, do not police which library gets imported. Choose whatever
is idiomatic and correct for the runtime.

### 4.5 No mutable static or global state

Script instances are reused across message exchanges. A static field, or a
variable declared outside `processData` and mutated inside it, may carry data from
one message into the next. This produces intermittent, load-dependent
cross-contamination that is close to impossible to reproduce. **All state lives in
local variables or in exchange properties** that are set outside the script.

### 4.6 Do not swallow exceptions

The default behaviour on bad data is to throw. The iFlow's Exception Subprocess
exists to handle it, and Message Monitoring shows the failure. A `try/catch` that
logs and returns `message` unchanged turns a failure into a silent success, which
is the worst outcome available.

Catch only to add context, then rethrow:

```groovy
// --- Validate ---------------------------------------------------------
// A missing order ID cannot be recovered here; fail so the exception
// subprocess can handle it and the message shows as failed in monitoring.
if (!orderId) {
    message.setProperty('errorReason', 'OrderId missing in incoming payload')
    throw new Exception('OrderId missing in incoming payload')
}
```

---

## 5. Use libraries, do not hand-roll

Anything with edge cases that a standard library already solved is written with
that library. Hand-rolled versions look correct against the sample and are wrong
in production.

| Task | Hand-rolled version that is wrong | Do instead |
| --- | --- | --- |
| Timezone conversion | adding a fixed number of hours | `java.time` with an explicit `ZoneId` |
| Date formatting or parsing | string slicing | `DateTimeFormatter` |
| URL encoding | replacing spaces with `%20` or `+` by hand | a URL-encoding library, aware of the difference between form, query and path encoding |
| Base64 | manual table or string juggling | the platform Base64 encoder/decoder |
| Charset conversion | character-by-character replacement of accented letters | decode bytes with the source charset, encode with the target |
| XML escaping | replacing `&`, `<`, `>` by hand | the XML utility escape method, or let a builder do it |
| JSON building | string concatenation | a JSON builder or output class |
| CSV parsing | `split(',')` | a real CSV parse that respects quoting and embedded separators |
| Hashing, HMAC | anything homemade | `MessageDigest` / `Mac` |

Which specific class or library is up to the runtime and the situation. The rule
is that a library does it, not that a particular import appears.

Two traps that deserve naming because they look solved when they are not:

**URL encoding is context-dependent.** Form encoding turns a space into `+`;
path and query encoding want `%20`. Encoding a whole URL in one call also
destroys the `:` and `/` separators. Encode component by component, and say which
context the code assumes.

**`split(',')` is not CSV.** It breaks on the first quoted field containing a
comma, which real data always eventually contains.

---

## 6. Groovy idioms that behave differently here

### 6.1 Truthiness

Groovy truth covers null, empty string, empty collection and zero in one test.
Custom emptiness helpers are noise:

```groovy
// Correct
if (orderId) { ... }

// Unnecessary
if (orderId != null && orderId.trim() != '') { ... }
```

Two caveats:

A variable holding boolean `false`, integer `0`, or `BigDecimal.ZERO` is falsy,
so `if (flag)` cannot distinguish "unset" from "legitimately false". When a false
value is meaningful, test explicitly against `null`.

`.trim()` is still needed when the value may be whitespace only: a string of
three spaces is truthy.

### 6.2 GPath returns empty, not null

`xml.Missing.text()` returns `''`, and an empty `GPathResult` is falsy. So
`if (xml.Missing)` and `if (xml.Missing.text())` both behave sensibly, but
`xml.Missing.text() == null` is never true. Do not test XML absence against null.

To distinguish "element absent" from "element present but empty", use
`xml.Missing.isEmpty()` or check `size()`.

### 6.3 XML namespaces

`XmlSlurper` is namespace-aware by default, which is why real SAP payloads that
work in an online Groovy playground fail on the tenant. Three workable
approaches, in order of preference:

```groovy
// 1. Declare and use the prefix. Explicit and safe.
def xml = new XmlSlurper().parseText(body)
xml.declareNamespace(ns: 'urn:sap:example')
def id = xml.'ns:Header'.'ns:OrderId'.text()

// 2. Turn namespace awareness off when the prefixes carry no meaning
//    for this transformation. Arguments are (validating, namespaceAware).
def plain = new XmlSlurper(false, false).parseText(body)
def id2 = plain.Header.OrderId.text()

// 3. Depth-first search by local name, when the path may vary.
def id3 = xml.'**'.find { it.name() == 'OrderId' }?.text()
```

Never write `xml.'ns:Element'` without having declared `ns` first. It silently
matches nothing.

### 6.4 Slurper versus parser versus builder

| Need | Class |
| --- | --- |
| Read values, light in-place edits | `XmlSlurper` |
| Structural changes: add, remove, move nodes | `XmlParser` |
| Build a document from scratch | `MarkupBuilder` or `StreamingMarkupBuilder` |

After editing an `XmlSlurper` result, serialise with `XmlUtil.serialize(...)`.
The changes are lazy and only materialise on serialisation, which is why
inspecting the `GPathResult` after a `replaceBody` can look as though nothing
happened.

---

## 7. Recipes

### 7.1 Message log and attachments

```groovy
def Message processData(Message message) {
    def body = message.getBody(java.lang.String)
    def messageLog = messageLogFactory.getMessageLog(message)
    if (messageLog != null) {
        messageLog.addAttachmentAsString('My Attachment', body, 'text/plain')
    }
    return message
}
```

`messageLogFactory` is bound implicitly; it needs no import. Attachments consume
tenant storage and are visible in Message Monitoring, so attach full payloads
only on an error path or when the user has asked for it, and never attach
credentials or personal data by default.

### 7.2 Custom headers

Information from the payload can be stored in filterable variables that are known as "custom headers". The most common use case is storing functional data (e.g. invoice number, business partner identifier) in the logs so that a user can later look for specific executions. Custom headers are different from message headers. Example:

```groovy
import com.sap.gateway.ip.core.customdev.util.Message;
import java.util.HashMap;

def Message processData(Message message) {
(...)
def messageLog = messageLogFactory.getMessageLog(message);
if(messageLog != null)
{
         messageLog.addCustomHeaderProperty("customHeaderName", "value")
}
}

```

### 7.3 Credentials

Read secrets from the Security Material, never from a literal in the script:

```groovy
def service = ITApiFactory.getService(SecureStoreService.class, null)
def cred    = service.getUserCredential('MY_CREDENTIAL_ALIAS')
def user    = cred.getUsername()
def pass    = cred.getPassword().toString()   // getPassword() returns char[]
```

This is for building a payload or a header that a later adapter consumes. It is
not licence to make the call from the script; section 4.1 still applies.

---

## 8. Anti-patterns

Each of these is something an LLM produces by default and a CPI runtime
punishes.

| Anti-pattern | Correct form |
| --- | --- |
| `body =~ /<OrderId>(.*?)<\/OrderId>/` | `new XmlSlurper().parseText(body).Header.OrderId.text()` |
| `import groovy.util.XmlSlurper` | `import groovy.xml.*` |
| `new URL(endpoint).text` | An adapter step; remove the call from the script |
| `message.setHeader('authToken', token)` | `message.setProperty('authToken', token)` |
| `message.getBody(String)` called twice | Read once into a local variable |
| `message.setBody(mapObject)` | `message.setBody(JsonOutput.toJson(mapObject))` |
| `new Date(now.time + 2*60*60*1000)` | `ZonedDateTime` with an explicit `ZoneId` |
| `value.replace(' ', '%20')` | A URL-encoding API, per component |
| `line.split(',')` on CSV | A CSV parse that handles quoting |
| `static Map cache = [:]` at script level | A local variable or an exchange property |
| `try { ... } catch (e) { return message }` | Rethrow, after setting an error property |
| `if (!s.isEmpty() && s != null)` | `if (s)` |
| `xml.'ns:Order'` with no `declareNamespace` | Declare the prefix, or parse with namespace awareness off |
| Hardcoded user and password literals | Security Material via the secure store service |

---

## 9. What to deliver

A finished answer is more than the script. Return:

1. **The script**, complete and runnable, following section 2.
2. **The contract**: which properties and headers it expects to exist, and which
   it sets. One or two lines; the header comment block already carries the
   detail.
3. **Placement**: where the Script step goes in the iFlow, and any step that must
   precede it (a Content Modifier setting a property, a converter, and so on).
4. **Assumptions**, if any question in section 1 went unanswered.

Do not pad the answer with a line-by-line walkthrough of code that is already
commented.

---

## 10. Check before returning

- [ ] Signature is `def Message processData(Message message)` and it returns `message`.
- [ ] `com.sap.gateway.ip.core.customdev.util.Message` is imported.
- [ ] Description block sits between the imports and the function, and lists the properties and headers read and written.
- [ ] Block comments mark the phases; no comment pushes a line past ~100 characters.
- [ ] No `groovy.util.` anywhere. XML imports use `import groovy.xml.*`.
- [ ] No regular expression is parsing XML or JSON structure.
- [ ] No HTTP, JDBC, JMS, SFTP, mail or URL call.
- [ ] Nothing in the script duplicates a palette step without the user having agreed to it.
- [ ] Every timestamp uses a zone the user specified, handled by a date-time library.
- [ ] Encoding, Base64, URL escaping and CSV go through libraries, not string surgery.
- [ ] Sensitive or purely internal values are in properties, not headers.
- [ ] The body is read once.
- [ ] `setBody` receives a String, `byte[]` or `InputStream`.
- [ ] No static or script-level mutable state.
- [ ] Errors throw rather than being swallowed.
- [ ] No credentials in literals.
- [ ] Emptiness tests use Groovy truth, with an explicit null test wherever `false` or `0` is a meaningful value.
