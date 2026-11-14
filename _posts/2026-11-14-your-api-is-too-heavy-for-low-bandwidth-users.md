---
title: Your API Is Too Heavy for Low-Bandwidth Users
date: 2026-11-14 22:30:00 +0530
categories: [Backend, API Design]
tags: [protobuf, protocol-buffers, rest-api, performance, python, mobile-games]
---




# Your API Is Too Heavy for Low-Bandwidth Users



Your server is fast. Your code is clean. Your tests pass. And yet a user on a shaky connection is staring at a loading spinner and slowly losing the will to live.

The problem isn't your server. It's that you're mailing a sofa when you could be mailing a postcard.\

![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/d32e0df3-7f25-4562-ab66-bce10753a20f.png align="center")

This post covers why that happens, how to fix it, and runnable code to prove it, so you can win arguments in code review.

---

## Step 0: To send data anywhere, you must turn it into bytes

Files, networks, databases: they all speak bytes. Turning your data into bytes is called **encoding**, and the format you pick decides how big, how fast, and how painful life gets. There are three families, and they get progressively less embarrassing.

![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/4ae74d4c-fb32-408f-aba3-3ec540aa6986.png align="center")

### Family 1: Language-specific formats (a.k.a. "it worked on my machine")

Java has `Serializable`. Python has `pickle`. Ruby has `Marshal`. They're built in, they're one line of code, and they're a trap:

- **Language lock-in.** Write data in Java, try to read it in Python or Go, and enjoy your new hobby: suffering. You're married to that language now.
- **Security nightmare.** Unpickling untrusted data can run arbitrary code. It's the "just click this email attachment" of serialization.
- **Versioning pain.** Change your class and old data may explode.
- **Meh efficiency.** Java's built-in serialization is famously slow and bloated.

  ![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/700f145e-9926-40db-a361-ac3a4f07e656.png align="center")

Use these for short-lived, in-process stuff only. Not for anything you'd want to read next Tuesday.

### Family 2: JSON, XML, CSV (the universal translators that charge by the byte)

Human-readable, supported everywhere, and the reason half the internet works. But they have quirks:

- **CSV** has no types. Is `123` a number or a string? Nobody knows. Not even CSV.
- **XML** treats everything as text unless you add a schema, and then you have *two* problems.
- **JSON** knows strings from numbers, but the spec has just one "number" type, so ints and floats are the same to it. Also, integers above 2⁵³ quietly lose precision in JavaScript. Your user ID may be someone else's user ID. Fun!
- All three repeat every field name in every record and write numbers as text. `"player_id"` is typed out in full, every single time, like a politician repeating a slogan.

  ![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/5510976d-859b-4ffb-ab37-70821a33f6e8.png align="center")

For small data this is fine. At terabyte scale, or on a slow network, it hurts.

### Family 3: Binary encoding with a schema (the grown-up in the room)

If the data is used inside your organization, or bandwidth and latency matter, you don't have to use the lowest-common-denominator format. Pick something compact and fast. **Protocol Buffers**, born at Google, is the big name here.

You write a schema, a code generator makes classes for Python, Go, Java, C#, and more, and your app just calls the generated code to encode and decode. Same schema, every language. Data written in one is readable in all.

![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/4539f57b-b5e3-499e-92ac-f6e882e2d1b8.png align="center")

---

## Let's define a schema

Scenario: a player submits a score in a mobile game.

```proto
// game.proto
syntax = "proto3";
package game;

message Item {
  uint32 item_id  = 1;
  uint32 quantity = 2;
}

message ScoreUpdate {
  uint64 player_id = 1;
  string username  = 2;
  uint32 level     = 3;
  uint32 score     = 4;
  float  accuracy  = 5;
  uint64 timestamp = 6;
  repeated Item items = 7;
}
```

Generate the Python code:

```bash
pip install protobuf grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. game.proto
```

You now have `game_pb2.py`. See those little `= 1`, `= 2` numbers? On the wire they replace the field names. "player_id" (9 characters) becomes "1" (one byte, tag included). That's most of the magic.

---

## The showdown: JSON vs CSV vs Pickle vs Protobuf

```python
# compare_sizes.py
import csv, io, json, pickle
import game_pb2

data = {
    "player_id": 482910, "username": "aarav_k", "level": 27,
    "score": 154300, "accuracy": 0.87, "timestamp": 1731571200,
    "items": [
        {"item_id": 101, "quantity": 3},
        {"item_id": 205, "quantity": 1},
        {"item_id": 330, "quantity": 12},
    ],
}

# JSON
json_bytes = json.dumps(data).encode()

# CSV (flattened; nested items get squeezed into one column, as CSV intended)
buf = io.StringIO()
w = csv.writer(buf)
w.writerow(data.keys())
w.writerow([
    data["player_id"], data["username"], data["level"], data["score"],
    data["accuracy"], data["timestamp"],
    ";".join(f'{i["item_id"]}:{i["quantity"]}' for i in data["items"]),
])
csv_bytes = buf.getvalue().encode()

# Protobuf
msg = game_pb2.ScoreUpdate(
    player_id=data["player_id"], username=data["username"],
    level=data["level"], score=data["score"],
    accuracy=data["accuracy"], timestamp=data["timestamp"],
)
for item in data["items"]:
    msg.items.add(item_id=item["item_id"], quantity=item["quantity"])
proto_bytes = msg.SerializeToString()

print(f"JSON      : {len(json_bytes)} bytes")
print(f"CSV       : {len(csv_bytes)} bytes")
print(f"Pickle    : {len(pickle.dumps(data))} bytes")
print(f"Protobuf  : {len(proto_bytes)} bytes")
print(f"Savings vs JSON: {100 * (1 - len(proto_bytes) / len(json_bytes)):.1f}%")
```

The real output, run on my machine, not made up for dramatic effect:

| Format | Size (bytes) |
| --- | --- |
| JSON | 228 |
| CSV | 118 |
| Pickle | 180 |
| **Protobuf** | **50** |

That's about **78% smaller** than JSON.

![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/9d5d19c9-6fd3-4da4-8a2e-70e869d76a20.png align="center")

A fair warning, since I like being trusted: in production our average was **~60%**, not 78%. Real payloads have more strings, and strings don't shrink as much as numbers. Your mileage will vary. Measure your own endpoints.

**"But I'll just gzip my JSON!"** Good instinct. gzip took JSON from 228 to 159 bytes. But for payloads this small, gzip adds overhead (Protobuf went 50 to 70 gzipped). On big responses, gzip and Protobuf stack nicely. They're teammates, not rivals.

---

## Proving it with a POST test

First, an endpoint that accepts both formats via content negotiation. Same URL, same REST, only the body changes:

```python
# app.py
from flask import Flask, request, jsonify
from google.protobuf.json_format import ParseDict, MessageToDict
import game_pb2

app = Flask(__name__)

@app.post("/scores")
def submit_score():
    msg = game_pb2.ScoreUpdate()
    if request.mimetype == "application/x-protobuf":
        msg.ParseFromString(request.get_data())
    else:
        ParseDict(request.get_json(), msg)
    # ... validate and save msg ...
    return jsonify(MessageToDict(msg)), 201
```

Now the tests (`pip install pytest flask protobuf`):

```python
# test_payloads.py
import json
import pytest
import game_pb2
from app import app

PAYLOAD = {
    "player_id": 482910, "username": "aarav_k", "level": 27,
    "score": 154300, "accuracy": 0.87, "timestamp": 1731571200,
    "items": [{"item_id": 101, "quantity": 3}, {"item_id": 205, "quantity": 1}],
}

@pytest.fixture
def client():
    return app.test_client()

def build_proto():
    msg = game_pb2.ScoreUpdate(**{k: v for k, v in PAYLOAD.items() if k != "items"})
    for item in PAYLOAD["items"]:
        msg.items.add(**item)
    return msg.SerializeToString()

def test_json_post(client):
    body = json.dumps(PAYLOAD).encode()
    res = client.post("/scores", data=body, content_type="application/json")
    assert res.status_code == 201

def test_protobuf_post(client):
    body = build_proto()
    res = client.post("/scores", data=body, content_type="application/x-protobuf")
    assert res.status_code == 201

def test_protobuf_payload_is_smaller():
    json_size = len(json.dumps(PAYLOAD).encode())
    proto_size = len(build_proto())
    print(f"JSON={json_size}B  Protobuf={proto_size}B")
    assert proto_size < json_size * 0.5   # expect >50% saving

def test_roundtrip_preserves_data():
    msg = game_pb2.ScoreUpdate()
    msg.ParseFromString(build_proto())
    assert msg.player_id == PAYLOAD["player_id"]
    assert msg.username == PAYLOAD["username"]
    assert len(msg.items) == len(PAYLOAD["items"])
```

Run `pytest -s test_payloads.py` and watch the byte counts print. It's oddly satisfying.

Want to test it over real HTTP? Fire both formats with `curl` and compare the `Content-Length` headers:

```bash
curl -X POST localhost:5000/scores -H "Content-Type: application/json" -d @score.json -v
curl -X POST localhost:5000/scores -H "Content-Type: application/x-protobuf" --data-binary @score.bin -v
```

---

## Bonus: Protobuf as database compression

Protobuf isn't only for networks. Store records as compact blobs:

```python
import sqlite3
import game_pb2

db = sqlite3.connect("game.db")
db.execute("""CREATE TABLE IF NOT EXISTS scores (
    player_id INTEGER, level INTEGER, payload BLOB)""")
db.execute("CREATE INDEX IF NOT EXISTS idx_player ON scores(player_id)")

msg = game_pb2.ScoreUpdate(player_id=482910, username="aarav_k", level=27, score=154300)
db.execute("INSERT INTO scores VALUES (?, ?, ?)",
           (msg.player_id, msg.level, msg.SerializeToString()))

row = db.execute("SELECT payload FROM scores WHERE player_id = ?", (482910,)).fetchone()
loaded = game_pb2.ScoreUpdate()
loaded.ParseFromString(row[0])
```

The `.proto` file acts as the contract for reading and writing the blob, and field numbers keep old rows readable as the schema evolves. Add a new field, and old rows still decode, with the new field at its default value.

**Important plot twist:** the schema is *not* a database index. If you want to filter or sort by something, keep it in a normal indexed column (like `player_id` and `level` above) and let the rest live in the blob. Don't try to `WHERE` your way into a binary blob. That's how you end up on a forum at 2 a.m.

![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/1f97174d-9cca-41f7-9c28-8067b4d7fc24.png align="center")

---

## Why mobile games love this

Mobile games send tiny payloads constantly: score updates, inventory syncs, matchmaking state, leaderboard fetches. Players are often on a wobbly 3G connection in a train tunnel, where latency is the real boss fight. Every saved byte means fewer retries, fewer timeouts, less mobile data burned, and less battery spent on the radio.

Where it helps in a game:

- **Save-game sync:** upload progress as one compact blob, not a JSON novel.
- **Leaderboards:** fetch the top 100 in a fraction of the bytes.
- **Real-time state:** send positions and actions in tight binary messages.
- **Offline-first caching:** keep the last known state on-device as Protobuf, saving disk and load time.

  ![](https://cdn.hashnode.com/uploads/covers/6ae1ac48aeab3c99f717886e/57cb22b1-be5c-4884-9655-ee73aa86aa3b.png align="center")

---

## The fine print (because honesty is a feature)

- **Debugging is harder.** You can't just read a binary payload in the network tab. Log decoded messages in dev, and keep a JSON fallback for the same endpoint.
- **Never reuse a field number.** Ever. Mark deleted fields as `reserved`. Reusing a number is how old clients start interpreting your score as a username.
- **JSON is still great** for tiny datasets and public APIs where humans need to read the thing. Don't be a Protobuf evangelist at a lemonade stand.

---

## TL;DR

- Language-specific formats lock you in and can be unsafe.
- JSON, XML, and CSV are universal but heavy.
- Schema-based binary formats like Protocol Buffers give you small payloads, cross-language support, and safe schema evolution.
- Choosing a data format is a performance decision. If your users are on slow networks, this is one of the cheapest wins you can ship.

*Found this useful? Share it with the teammate who still uses* `pickle` *in production. They'll know who they are.*