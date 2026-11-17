# Issue Resolution: Multi-Value Tornado Headers Are Fetched with get() Rather Than get_list()

## 1. Upstream Metadata
- **Repository:** `marshmallow-code/webargs`
- **Issue Reference:** [#1109](https://github.com/marshmallow-code/webargs/issues/1109)
- **Title:** `Multi-value tornado headers are fetched with get() rather than get_list()`
- **Component:** `src/webargs/tornadoparser.py` (`WebArgsTornadoMultiDictProxy`), `src/webargs/multidictproxy.py` (`MultiDictProxy`)

---

## 2. Problem Description & Root Cause
In Tornado, request headers are instances of `tornado.httputil.HTTPHeaders`. When multiple header values are added for the same key:
```python
headers = tornado.httputil.HTTPHeaders()
headers.add("X-Tag", "foo")
headers.add("X-Tag", "bar")
```
- Calling `headers.get("X-Tag")` returns the comma-joined string `'foo,bar'`.
- Calling `headers.get_list("X-Tag")` returns the actual list of values `['foo', 'bar']`.

### Root Cause:
1. In `WebArgsTornadoMultiDictProxy.__getitem__`, when retrieving values for fields listed in `self.multiple_keys` (such as `fields.List`), the proxy called `value = self.data.get(key, core.missing)` and then iterated over `value` with `for v in value`.
2. When `self.data` was an `HTTPHeaders` instance, `value` was the joined string `'foo,bar'`. Iterating over this string yielded individual characters (`['f', 'o', 'o', ',', 'b', 'a', 'r']`) instead of the distinct header values (`['foo', 'bar']`).
3. Furthermore, `MultiDictProxy` supported `getlist` and `getall`, but omitted `get_list` used by Tornado's `HTTPHeaders`.

---

## 3. Reproduction Test
A test `test_tornado_httpheaders_multiple_values` was added to `tests/test_tornadoparser.py`:
```python
def test_tornado_httpheaders_multiple_values(self):
    headers = tornado.httputil.HTTPHeaders()
    headers.add("works", "Prometheus Bound")
    headers.add("works", "Prometheus Unbound")
    request = make_request(headers=headers)
    result = parser.load_headers(request, author_schema)
    assert result["works"] == ["Prometheus Bound", "Prometheus Unbound"]
```
Prior to the fix, this test failed with:
```
AssertionError: assert ['P', 'r', 'o', 'm', 'e', 't', 'h', 'e', 'u', 's', ' ', 'B', 'o', 'u', 'n', 'd', ',', 'P', 'r', 'o', 'm', 'e', 't', 'h', 'e', 'u', 's', ' ', 'U', 'n', 'b', 'o', 'u', 'n', 'd'] == ['Prometheus Bound', 'Prometheus Unbound']
```

---

## 4. Code Fix
1. In `src/webargs/tornadoparser.py`:
   Within `WebArgsTornadoMultiDictProxy.__getitem__`, checked `if hasattr(self.data, "get_list"): value = self.data.get_list(key)` when `key in self.multiple_keys` before decoding values.
2. In `src/webargs/multidictproxy.py`:
   Added `if hasattr(self.data, "get_list"): return self.data.get_list(key)` in `MultiDictProxy.__getitem__` alongside existing `getlist` and `getall` checks.

---

## 5. Verification Results
- **Reproduction Test:** `1 passed, 33 deselected, 2 warnings in 0.16s` (`tests/test_tornadoparser.py -k test_tornado_httpheaders_multiple_values`).
- **Tornado Parser Test Suite:** `34 passed, 66 warnings in 0.24s` (`tests/test_tornadoparser.py`).
- **Parsers Suite:** `328 passed, 2 skipped, 4 warnings in 3.36s` (`test_core.py`, `test_flaskparser.py`, `test_djangoparser.py`, `test_bottleparser.py`).
- **Linter Check:** `ruff check` on modified files passed with zero errors.
- **Repository Commit:** Pushed cleanly to `rajanjava/webargs-private` on branch `main`.
