---
title: RPC Interface
nav_order: 4
---

<!---
  Copyright 2020 Deephaven Data Labs

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

RPC Interface
=============

Barrage is an RPC interface for high-performance data services based on Arrow,
for ticking data sets built on top of gRPC.

Barrage is an extension of Apache Arrow Flight. The extension works by sending
additional metadata via the `app_metadata` field on `FlightData`. This metadata
is used to communicate the necessary additional information between server
and client. These types are flatbuffers, so that we may more easily lift the
`app_metadata` into the `RecordBatch` flatbuffer once Arrow supports byte-array
metadata, at that layer.

The main subscription mechanism is initiated via a `DoExchange`. The client
sends a SubscriptionRequest (or as many as they like) and the server sends
barrage updates to satisfy their subscription's requirements.


Flat Buffer Definitions
-----------------------

```fbs
// Copyright 2020 Deephaven Data Labs
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

namespace io.deephaven.barrage.flatbuf;

enum BarrageMessageType : byte {
  /// A barrage message wrapper might send a None message type
  /// if the msg_payload is empty. This is helpful for browser clients
  /// that must send all client-streaming messages out-of-band
  /// (note: browsers do not support client-streaming gRPCs)
  None = 0,

  /// for session management
  NewSessionRequest = 1,
  RefreshSessionRequest = 2,
  SessionInfoResponse = 3,

  /// for subscription parsing/management (aka DoGet, DoPush, DoExchange)
  BarrageSerializationOptions = 4,
  BarrageSubscriptionRequest = 5,
  BarrageUpdateMetadata = 6,

  // enum values greater than 127 are reserved for custom client use
}

/// The message wrapper used for all barrage app_metadata fields.
table BarrageMessageWrapper {
  /// Used to identify this type of app_metadata vs other applications.
  /// The magic value is '0x6E687064'. It is the numerical representation of the ASCII "dhvn".
  magic: uint;

  /// The msg type being sent.
  msg_type: BarrageMessageType;

  /// The msg payload.
  msg_payload: [byte];

  /// Simulated Client Streaming Parameters (not required for regular client-only / bidirectional streams)

  /// this ticket is used to find the stream in the session
  rpc_ticket: [byte];

  /// if messages are received out of sequence they get queued
  sequence: long;

  /// after processing this message tell the server to close the stream and to release the rpc_ticket
  half_close_after_message: boolean;
}

/// Establish a new session.
table NewSessionRequest {
  /// A nested protocol version (gets delegated to handshake)
  protocol_version: uint;

  /// Arbitrary auth/handshake info.
  payload: [byte];
}

/// Refresh the provided session.
table RefreshSessionRequest {
  /// this session token is only required if it is the first request of a handshake rpc stream
  session: [byte];
}

/// Information about the current session state.
table SessionInfoResponse {
  /// this is the metadata header to identify this session with future requests; it must be lower-case and remain static for the life of the session
  metadata_header: [byte];

  /// this is the session_token; note that it may rotate
  session_token: [byte];

  /// a suggested time for the user to refresh the session if they do not do so earlier; value is denoted in milliseconds since epoch
  token_refresh_deadline_ms: long;
}

/// There will always be types that cannot be easily supported over IPC. These are the options:
///   Stringify (default) - Pretend the column is a string when sending over Arrow Flight (default)
///   JavaSerialization   - Use java serialization; the client is responsible for the deserialization
///   ThrowError          - Refuse to send the column and throw an internal error sharing as much detail as possible
enum ColumnConversionMode : byte { Stringify = 1, JavaSerialization, ThrowError }

table BarrageSerializationOptions {
  /// see enum for details
  column_conversion_mode: ColumnConversionMode = Stringify;

  /// Deephaven reserves a value in the range of primitives as a custom NULL value. This enables more efficient transmission
  /// by eliminating the additional complexity of the validity buffer.
  use_deephaven_nulls: bool;
}

/// Describes the subscription the client would like to acquire.
table BarrageSubscriptionRequest {
  /// Ticket for the source data set.
  ticket: [byte];

  /// The bitset of columns to subscribe to. An empty bitset unsubscribes from all columns.
  columns: [byte];

  /// This is an encoded and compressed Index of rows in position-space to subscribe to.
  viewport: [byte];

  /// Explicitly set the update interval for this subscription. Note that subscriptions with different update intervals
  /// cannot share intermediary state with other subscriptions and greatly increases the footprint of the non-conforming subscription.
  /// Setting this field does not guarantee updates will be received at this frequency.
  ///
  /// Note: if not supplied (default of zero) then the server uses a consistent value to be efficient and fair to all clients
  update_interval_ms: long;

  /// Optionally enable/disable special serialization features.
  serialization_options: BarrageSerializationOptions;
}

/// Holds all of the index data structures for the column being modified.
table BarrageModColumnMetadata {
  /// This is an encoded and compressed Index of rows for this column (within the viewport) that were modified.
  /// There is no notification for modifications outside of the viewport.
  modified_rows: [byte];
}

/// A data header describing the shared memory layout of a "record" or "row"
/// batch for a ticking barrage table.
table BarrageUpdateMetadata {
  /// The number of record batches that describe rows added (may be zero).
  num_add_batches: ushort;

  /// The number of record batches that describe rows modified (may be zero).
  num_mod_batches: ushort;

  /// This batch is generated from an upstream table that ticks independently of the stream. If
  /// multiple events are coalesced into one update, the server may communicate that here for
  /// informational purposes.
  first_seq: long;
  last_seq: long;

  /// Indicates if this message was sent due to upstream ticks or due to a subscription change.
  is_snapshot: bool;

  /// If this is a snapshot and the subscription is a viewport, then the effectively subscribed viewport
  /// will be included in the payload. It is an encoded and compressed Index.
  effective_viewport: [byte];

  /// If this is a snapshot, then the effectively subscribed column set will be included in the payload.
  effective_column_set: [byte];

  /// This is an encoded and compressed Index of rows that were added in this update.
  added_rows: [byte];

  /// This is an encoded and compressed Index of rows that were removed in this update.
  removed_rows: [byte];

  /// This is an encoded and compressed IndexShiftData describing how the keyspace of unmodified rows changed.
  shift_data: [byte];

  /// This is an encoded and compressed Index of rows that were included with this update.
  /// (the server may include rows not in addedRows if this is a viewport subscription to refresh
  ///  unmodified rows that were scoped into view)
  added_rows_included: [byte];

  /// The list of modified column data are in the same order as the field nodes on the schema.
  nodes: [BarrageModColumnMetadata];
}
```
