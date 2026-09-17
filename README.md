# Open-source case studies

Notes on bugs I reproduced, traced to a root cause, and fixed upstream. Every entry below is a merged pull request — the link goes to the PR, and the write-up is the part that doesn't fit in a diff.

The pattern is the same each time: make the failure happen reliably, find out why, change as little as possible, and lock the behaviour down with a test.

---

## 1. versitygw — an S3 bucket answered with an empty `<Rule/>`

**[versity/versitygw#2399](https://github.com/versity/versitygw/pull/2399)** · merged · Go · ★ 3k · `+126 / −6`

**Symptom.** `GET /<bucket>?object-lock` on a bucket that had object lock enabled but no default retention returned:

```xml
<ObjectLockConfiguration>
  <ObjectLockEnabled>Enabled</ObjectLockEnabled>
  <Rule></Rule>
</ObjectLockConfiguration>
```

AWS omits `<Rule>` entirely in that case, and clients validating against the S3 schema rejected the response.

**Root cause.** `ParseBucketLockConfigurationOutput` always set `Rule`. The response is marshalled with `encoding/xml` straight from the AWS SDK type, so a non-nil `Rule` with a nil `DefaultRetention` rendered as the empty element.

**Fix.** Set `Rule` only when a default retention exists. The path that does have retention is untouched.

**Validation.** A table test over the stored JSON asserts the marshalled XML for both cases — it reproduces the reported body before the change and passes after. `go test -race -tags=github ./auth/` passes, `go vet` and `gofmt -s` are clean.

---

## 2. jc — `http-headers` crashed on a whitespace-only line

**[kellyjonbrazil/jc#753](https://github.com/kellyjonbrazil/jc/pull/753)** · merged · Python · ★ 8.7k · `+35 / −1`

**Symptom.** Parsing input with a line containing only spaces raised `IndexError` in `http_headers`. `curl_head` reached the same code through `http_headers.parse`, so both entry points were affected.

**Root cause.** The parser dropped blank lines with `filter(None, ...)`, which removes empty strings but keeps whitespace-only ones. The next step called `line.split(maxsplit=1)[0]` on that line, and an empty split result has no index `0`.

**Fix.** Filter whitespace-only lines, matching what `hosts` already does after an earlier fix.

**Validation.** Two regression tests — one for a request, one for a response containing a whitespace-only line. Both raise `IndexError` without the change and pass with it.

---

## 3. django-ninja — async authentication callbacks ran twice

**[vitalik/django-ninja#1755](https://github.com/vitalik/django-ninja/pull/1755)** · merged · Python · ★ 9.2k · `+31 / −1`

**Symptom.** An async-marked authentication callback on a synchronous endpoint executed twice, and the run emitted both an un-awaited coroutine warning and an `async_to_sync` warning. Duplicated synchronous work inside `__call__` is the visible damage; the warnings are the clue.

**Root cause.** The synchronous authentication path called the callback once to inspect its return value. When that value was a coroutine it discarded it, then invoked the callback a second time through `async_to_sync`.

**Fix.** Await the coroutine returned by the first invocation instead of calling the callback again.

**Validation.** A new regression test covers async bearer authentication on a synchronous endpoint and checks the callback runs once with no warnings. `tests/test_auth_async.py` — 8 passed, `tests/test_auth.py` — 34 passed, plus clean `ruff format --check`, `ruff check`, and `mypy`.

---

More of the same lives on my GitHub profile: [@tunglambk](https://github.com/tunglambk).

If you maintain a project and hit something that looks like one of these, [email me](mailto:lamphambatung96@gmail.com).
