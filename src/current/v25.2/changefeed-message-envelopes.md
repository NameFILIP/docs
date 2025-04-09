---
title: Changefeed Message Envelope
summary: Learn how to configure the changefeed message envelope.
toc: true
---

In CockroachDB changefeeds, the _envelope_ is the structure of each [_message_]({% link {{ page.version.version }}/changefeed-messages.md %}). Changefeeds package _events_, triggered by an update to a row in a [watched table]({% link {{ page.version.version }}/change-data-capture-overview.md %}#watched-table), into messages according to the configured envelope. By default, changefeed messages use the `wrapped` envelope, which includes the primary key of the changed row and a top-level field indicating the value of the row after the change event (an "after" field with the new row values, or null for deletes).​ 

You can use changefeed options to customize what information the message will contain for to integrate with your downstream requirements. For example, the previous state of the row, the information of the cluster that the message is coming from, or the schema of the event payload in the message.

The possible envelope fields support use cases such as:

- 
- Routing events based on operation type.


On this page, review:

- [Use case](#use-cases) examples
- Reference lists:
    - The [options](#option-reference) to configure the envelope.
    - The supported envelope [fields](#field-reference).

{{site.data.alerts.callout_info}}
You can also specify the _format_ of changefeed messages, such as Avro. For more details, refer to [Message formats]({% link {{ page.version.version }}/changefeed-messages.md %}#message-formats).
{{site.data.alerts.end}}

## Overview



For a full reference of sink support, 

{% comment  %}Add intro to use cases here and the schema / source payload configurability{% endcomment %}

{% comment  %}
. This is useful for broadcasting primary key changes efficiently if downstream consumers only need to know which keys changed.
row: Sends the row’s new content directly as a flat JSON object, without any additional CDC metadata fields​
COCKROACHLABS.COM
. This envelope is simpler and omits fields like "after" or "updated".
bare: Similar to row, but any metadata (timestamps, keys, etc.) is nested under a special "__crdb__" field instead of top-level. The bare envelope places the row’s columns at the top level of the message (instead of under "after"), and is the default for sinkless changefeeds (CDC queries)​
COCKROACHLABS.COM
.
enriched: Provides an “enriched” message that can include additional metadata sections, such as information about the source cluster/node and the database schema for the change event. This envelope extends the default message structure with optional fields (configured via enriched_properties) to facilitate downstream processing that needs to know where the change came from or how to interpret the data (e.g., schema versioning). Enriched envelopes are typically used with sinks like Kafka or cloud sinks that can carry JSON or Avro payloads, and are not supported for sinkless feeds. (This envelope type was introduced in CockroachDB v25.2​
COCKROACHLABS.COM
.)
In addition to choosing an envelope type, you can use various envelope-related options (specified in the WITH clause of CREATE CHANGEFEED) to include or modify certain fields in the emitted messages. For example, the diff option adds a “before” image of changed rows, and the updated option adds a timestamp indicating when the row was updated. These options are often used in conjunction with the default wrapped envelope but may not be applicable to all envelope types (details below).
{% endcomment %}

## Use cases

The use case examples in the following sections emit to a Kafka sink. Use the [reference](#ref) sections to understand the options and envelope fields supported by different sinks.

use the table schema:

~~~sql
CREATE TABLE public.products (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    description STRING NULL,
    price DECIMAL(10,2) NOT NULL,
    in_stock BOOL NULL DEFAULT true,
    category STRING NULL,
    created_at TIMESTAMP NULL DEFAULT current_timestamp():::TIMESTAMP,
    CONSTRAINT products_pkey PRIMARY KEY (id ASC)
);
~~~
~~~sql
CREATE TABLE public.orders (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    customer_name STRING NOT NULL,
    email STRING NOT NULL,
    product_id UUID NOT NULL,
    quantity INT8 NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    order_date TIMESTAMP NULL DEFAULT current_timestamp():::TIMESTAMP,
    status STRING NULL DEFAULT 'pending':::STRING,
    CONSTRAINT orders_pkey PRIMARY KEY (id ASC),
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES public.products(id),
    CONSTRAINT check_quantity CHECK (quantity > 0:::INT8)
);
~~~

{{site.data.alerts.callout_info}}
The values that the `envelope` option accepts are compatible with different [changefeed sinks]({% link {{ page.version.version }}/changefeed-sinks.md %}), and the structure of the message will vary depending on the sink.
{{site.data.alerts.end}}

### Enable full fidelity message envelopes

A _full fidelity_ changefeed message envelope ensures complete information about every change event in the [watched tables]—including the before and after state of a row, timestamps, and rich metadata like source and schema information. This type of configured envelope allows downstream consumers to replay, audit, debug, or analyze changes with no loss of context.

Use the `envelope=enriched, enriched_properties='source, schema', diff` options with `CREATE CHANGEFEED` to create a full fidelity envelope:

~~~json
tbd
~~~


### Route events based on operation type

You may want to route change events in a table based on the operation type (insert, update, delete), which can be useful for correctly applying or handling a change in a downstream system or replication pipeline. Use the `envelope=enriched` option with `CREATE CHANGEFEED` to include the `op` field in the envelope:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE CHANGEFEED FOR TABLE products INTO 'external://kafka:9092' WITH envelope=enriched;
~~~
~~~json
{
  "after": {
    "category": "Home & Kitchen",
    "created_at": "2025-04-01T17:55:46.812942",
    "description": "Adjustable LED desk lamp with touch controls",
    "id": "32856ed8-34d3-45a3-a449-412bdeaa277c",
    "in_stock": true,
    "name": "LED Desk Lamp",
    "price": 22.30
  },
  "op": "c",
  "ts_ns": 1743792394409866000
}
~~~

The `op` field can contain:

- `c` for inserts.
- `u` for updates.
- `d` for deletes.

### Add envelope schema fields

Adding the schema of the event payload to the message envelope allows you to:

- Handle schema changes in downstream processing systems that require field types.
- Detect and adapt to changes in the table schema over time.
- Correctly parse and cast the data to deserialize into a different format.
- Automatically generate or synchronize schemas in downstream systems.
- Verify critical fields are present and set up alerts based on this.

Use the `envelope=enriched, enriched_properties=schema` options with `CREATE CHANGEFEED` to include the `schema` top-level field and the schema fields and types:

{% include_cached copy-clipboard.html %}
~~~ sql
CREATE CHANGEFEED FOR TABLE products INTO 'external://kafka:9092' WITH envelope=enriched, enriched_properties=schema;
~~~
~~~json
{
  "payload": {
    "after": {
      "category": "Home & Kitchen",
      "created_at": "2025-04-01T17:55:46.812942",
      "description": "Ceramic mug with 350ml capacity",
      "id": "8320b051-3ff7-4aa8-9708-78142fde7e31",
      "in_stock": true,
      "name": "Coffee Mug",
      "price": 12.50
    },
    "op": "c",
    "ts_ns": 1745353048801146000
  },
  "schema": {
    "fields": [
      {
        "field": "after",
        "fields": [
          {
            "field": "id",
            "optional": false,
            "type": "string"
          },
          {
            "field": "name",
            "optional": false,
            "type": "string"
          },
          {
            "field": "description",
            "optional": true,
            "type": "string"
          },
          {
            "field": "price",
            "name": "decimal",
            "optional": false,
            "parameters": {
              "precision": "10",
              "scale": "2"
            },
            "type": "float64"
          },
          {
            "field": "in_stock",
            "optional": true,
            "type": "boolean"
          },
          {
            "field": "category",
            "optional": true,
            "type": "string"
          },
          {
            "field": "created_at",
            "name": "timestamp",
            "optional": true,
            "type": "string"
          }
        ],
        "name": "products.after.value",
        "optional": false,
        "type": "struct"
      },
      {
        "field": "ts_ns",
        "optional": false,
        "type": "int64"
      },
      {
        "field": "op",
        "optional": false,
        "type": "string"
      }
    ],
    "name": "cockroachdb.envelope",
    "optional": false,
    "type": "struct"
  }
}
~~~

### Preserve the origin of data

When you have multiple changefeeds running from your cluster, or your CockroachDB cluster is part of a multi-architecture system, it is useful to have metadata on the origin of changefeed data in the message envelope.

Use the `envelope=enriched, enriched_properties=source` options with `CREATE CHANGEFEED` to include the `source` top-level field that contains metadata for the origin cluster and the changefeed job:

~~~sql
CREATE CHANGEFEED FOR TABLE products, orders INTO 'kafka://localhost:9092' WITH envelope=enriched, enriched_properties=source;
~~~
~~~json
{
  "after": {
    "category": "Home & Kitchen",
    "created_at": "2025-04-01T17:55:46.812942",
    "description": "Adjustable LED desk lamp with touch controls",
    "id": "32856ed8-34d3-45a3-a449-412bdeaa277c",
    "in_stock": true,
    "name": "LED Desk Lamp",
    "price": 26.30
  },
  "op": "c",
  "source": {
    "changefeed_sink": "kafka",
    "cluster_id": "3b38bd3f-af46-4083-9801-000000000000",
    "cluster_name": "",
    "db_version": "v25.2.0-alpha.1",
    "job_id": "1065892426841096193",
    "node_id": "1",
    "node_name": "localhost",
    "source_node_locality": "cloud=gce,region=us-east1,zone=us-east1-b"
  },
  "ts_ns": 1745429228563245000
}
~~~

For a sinkless changefeed,

~~~sql
CREATE CHANGEFEED FOR TABLE products, orders WITH envelope=enriched, enriched_properties=source;
~~~
~~~
{"key":"{\"id\": \"32856ed8-34d3-45a3-a449-412bdeaa277c\"}","table":"products","value":"{\"after\": {\"category\": \"Home \u0026 Kitchen\", \"created_at\": \"2025-04-01T17:55:46.812942\", \"description\": \"Adjustable LED desk lamp with touch controls\", \"id\": \"32856ed8-34d3-45a3-a449-412bdeaa277c\", \"in_stock\": true, \"name\": \"LED Desk Lamp\", \"price\": 22.30}, \"op\": \"c\", \"source\": {\"changefeed_sink\": \"sinkless buffer\", \"cluster_id\": \"3b38bd3f-af46-4083-9801-000000000000\", \"cluster_name\": \"\", \"db_version\": \"v25.2.0-alpha.1\", \"job_id\": \"0\", \"node_id\": \"3\", \"node_name\": \"localhost\", \"source_node_locality\": \"\"}, \"ts_ns\": 1745423277149449000}"}
~~~

### Audit changes in data

You can include both the previous and updated states of a row in the message envelope to support use cases like auditing or applying change-based logic in downstream systems. Use the `diff` option with `CREATE CHANGEFEED` to incluse the previous state of the row:

~~~sql
CREATE CHANGEFEED FOR TABLE products INTO 'kafka://localhost:9092' WITH diff;
~~~

The `diff` option adds the `before` field to the envelope containing the state of row before the change:

~~~json
{
  "after": {
    "category": "Home & Kitchen",
    "created_at": "2025-04-01T17:55:46.812942",
    "description": "Adjustable LED desk lamp with touch controls",
    "id": "32856ed8-34d3-45a3-a449-412bdeaa277c",
    "in_stock": true,
    "name": "LED Desk Lamp",
    "price": 26.30
  },
  "before": {
    "category": "Home & Kitchen",
    "created_at": "2025-04-01T17:55:46.812942",
    "description": "Adjustable LED desk lamp with touch controls",
    "id": "32856ed8-34d3-45a3-a449-412bdeaa277c",
    "in_stock": true,
    "name": "LED Desk Lamp",
    "price": 22.30
  }
}
~~~

For an insert into the table, the `before` field contains `null`:

~~~json
{
  "after": {
    "category": "Electronics",
    "created_at": "2025-04-23T13:48:40.981735",
    "description": "Over-ear headphones with active noise cancellation",
    "id": "3d8f4ca4-36e9-43b2-b057-d691624a4cba",
    "in_stock": true,
    "name": "Noise Cancelling Headphones",
    "price": 129.50
  },
  "before": null
}
~~~

## Option reference

For a full list of options that modify the message envelope, refer to the following table:

Option | Description | Sink support
-------+-------------+-------------+-------------
`diff` | Include a `"before"` field in each message, showing the state of the row before the change. Supported with `wrapped` or `enriched` envelopes. | All
`enriched_properties` | (Only applicable when `envelope=enriched` is set) Specify the type of metadata included in the message payload. Values:  `source`, `schema`. | Kafka, Pub/Sub, webhook, sinkless
`envelope=bare` | Emit an envelope without the `"after"` wrapper. The row's column data is at the top level of the message. Metadata that would typically be separate will be under a `"__crdb__"` field. Provides a more compact structure to the envelope. `bare` is the default envelope when using [CDC queries]({% link {{ page.version.version }}/cdc-queries.md %}). When `bare` is used with the Avro format, `record` will replace the `after` keyword.
`envelope=enriched` | Extend the envelope with additional metadata fields. With `enriched_properties`, includes a `"source"` field and/or a `"schema"` field with extra context. | Kafka, Pub/Sub, webhook, sinkless
`envelope=key_only` | Send only the primary key of the changed row and no value payload, which is more efficient if only the key of the changed row is needed. Not compatible with the `updated` option. | Kafka, sinkless
`envelope=row`  | Emit the row data without any additional metadata field in the envelope. Not supported in Avro format or with the `diff` option. | Kafka, sinkless
`envelope=wrapped` (default) | Produce changefeed messages in a wrapped structure with metadata and row data. `wrapped` includes an `"after"` field, and optionally a `"before"` field if `diff` is used. **Note:** Envelopes contain a primary key when your changefeed is emitting to a sink that does not have a message key as part of its protocol. By default, messages emitting to Kafka sinks do not have the primary key array, because the key is part of the message metadata. Use the `key_in_value` option to include a primary key array in messages emitted to Kafka sinks. | All
`full_table_name` | Use the fully qualified table name (`database.schema.table`) in topics, subjects, schemas, and record output instead of the default table name. Including the full table name prevents unintended behavior when the same table name is present in multiple databases. | 
`mvcc_timestamp` | Emit the MVCC timestamp for each change event. The message envelope contains the MVCC timestamp of the changed row, even during the changefeed's initial scan. Provides a precise database commit timestamp, which is useful for debugging or strict ordering.
`updated` | Add an `"updated"` timestamp field to each message, showing the commit time of the change. When the changefeed runs an initial scan or a schema change backfill, the `"updated"` field will reflect the time of the scan or backfill, not the MVCC timestamp. |

### Examples


- `wrapped`

    `wrapped` is the default envelope structure for changefeed messages. This envelope contains an array of the primary key (or the key as part of the message metadata), a top-level field for the type of message, and the current state of the row (or `null` for [deleted rows](#delete-messages)).

    The message envelope contains a primary key array when your changefeed is emitting to a sink that does not have a message key as part of its protocol, (e.g., cloud storage, webhook sinks, or Google Pub/Sub). By default, messages emitted to Kafka sinks do not have the primary key array, because the key is part of the message metadata. If you would like messages emitted to Kafka sinks to contain a primary key array, you can use the [`key_in_value`]({% link {{ page.version.version }}/create-changefeed.md %}#key-in-value) option. Refer to the following message outputs for examples of this.

    Cloud storage sink:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE vehicles INTO 'external://cloud';
    ~~~
    ~~~
    {"after": {"city": "seattle", "creation_time": "2019-01-02T03:04:05", "current_location": "86359 Jeffrey Ranch", "ext": {"color": "yellow"}, "id": "68ee1f95-3137-48e2-8ce3-34ac2d18c7c8", "owner_id": "570a3d70-a3d7-4c00-8000-000000000011", "status": "in_use", "type": "scooter"}, "key": ["seattle", "68ee1f95-3137-48e2-8ce3-34ac2d18c7c8"]}
    ~~~

    Kafka sink:

    Default when `envelope=wrapped` or `envelope` is not specified:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE vehicles INTO 'external://kafka';
    ~~~
    ~~~
    {"after": {"city": "washington dc", "creation_time": "2019-01-02T03:04:05", "current_location": "24315 Elizabeth Mountains", "ext": {"color": "yellow"}, "id": "dadc1c0b-30f0-4c8b-bd16-046c8612bbea", "owner_id": "034075b6-5380-4996-a267-5a129781f4d3", "status": "in_use", "type": "scooter"}}
    ~~~

    Kafka sink message with `key_in_value` provided:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE vehicles INTO 'external://kafka' WITH key_in_value, envelope=wrapped;
    ~~~
    ~~~
    {"after": {"city": "washington dc", "creation_time": "2019-01-02T03:04:05", "current_location": "46227 Jeremy Haven Suite 92", "ext": {"brand": "Schwinn", "color": "red"}, "id": "298cc7a0-de6b-4659-ae57-eaa2de9d99c3", "owner_id": "beda1202-63f7-41d2-aa35-ee3a835679d1", "status": "in_use", "type": "bike"}, "key": ["washington dc", "298cc7a0-de6b-4659-ae57-eaa2de9d99c3"]}
    ~~~

- `bare`

    `bare` removes the `after` key from the changefeed message and stores any metadata in a `crdb` field. When used with [`avro`](#avro) format, `record` will replace the `after` key.

    Cloud storage sink:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE vehicles INTO 'external://cloud' WITH envelope=bare;
    ~~~
    ~~~
    {"__crdb__": {"key": ["washington dc", "cd48e501-e86d-4019-9923-2fc9a964b264"]}, "city": "washington dc", "creation_time": "2019-01-02T03:04:05", "current_location": "87247 Diane Park", "ext": {"brand": "Fuji", "color": "yellow"}, "id": "cd48e501-e86d-4019-9923-2fc9a964b264", "owner_id": "a616ce61-ade4-43d2-9aab-0e3b24a9aa9a", "status": "available", "type": "bike"}
    ~~~

    {% include {{ page.version.version }}/cdc/bare-envelope-cdc-queries.md %}

    In CDC queries:

    A changefeed containing a `SELECT` clause without any additional options:

    ~~~sql
    CREATE CHANGEFEED INTO 'external://kafka' AS SELECT city, type FROM movr.vehicles;
    ~~~
    ~~~
    {"city": "los angeles", "type": "skateboard"}
    ~~~

    A changefeed containing a `SELECT` clause with the [`topic_in_value`]({% link {{ page.version.version }}/create-changefeed.md %}#topic-in-value) option specified:

    ~~~sql
    CREATE CHANGEFEED INTO 'external://kafka' WITH topic_in_value AS SELECT city, type FROM movr.vehicles;
    ~~~
    ~~~
    {"__crdb__": {"topic": "vehicles"}, "city": "los angeles", "type": "skateboard"}
    ~~~

- `key_only`

    `key_only` emits only the key and no value, which is faster if you only need to know the key of the changed row. This envelope option is only supported for [Kafka sinks]({% link {{ page.version.version }}/changefeed-sinks.md %}#kafka) or sinkless changefeeds.

    Kafka sink:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE users INTO 'external://kafka' WITH envelope=key_only;
    ~~~
    ~~~
    ["boston", "22222222-2222-4200-8000-000000000002"]
    ~~~

    {{site.data.alerts.callout_info}}
    It is necessary to set up a [Kafka consumer](https://docs.confluent.io/platform/current/clients/consumer.html) to display the key because the key is part of the metadata in Kafka messages, rather than in its own field. When you start a Kafka consumer, you can use `--property print.key=true` to have the key print in the changefeed message.
    {{site.data.alerts.end}}

    Sinkless changefeeds:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE users WITH envelope=key_only;
    ~~~
    ~~~
    {"key":"[\"seattle\", \"fff726cc-13b3-475f-ad92-a21cafee5d3f\"]","table":"users","value":""}
    ~~~

- `row`

    `row` emits the row without any additional metadata fields in the message. This envelope option is only supported for [Kafka sinks]({% link {{ page.version.version }}/changefeed-sinks.md %}#kafka) or sinkless changefeeds. `row` does not support [`avro`](#avro) format—if you are using `avro`, refer to the [`bare`](#bare) envelope option.

    Kafka sink:

    ~~~sql
    CREATE CHANGEFEED FOR TABLE vehicles INTO 'external://kafka' WITH envelope=row;
    ~~~
    ~~~
    {"city": "washington dc", "creation_time": "2019-01-02T03:04:05", "current_location": "85551 Moore Mountains Apt. 47", "ext": {"color": "red"}, "id": "d3b37607-1e9f-4e25-b772-efb9374b08e3", "owner_id": "4f26b516-f13f-4136-83e1-2ea1ae151c20", "status": "available", "type": "skateboard"}
    ~~~


## Field reference

CockroachDB provides multiple changefeed envelopes, each supported by different sinks and use cases.

The possible top-level fields in a JSON-formatted envelope:

- `payload`: The change event data. `payload` is a wrapper only with webhook sinks and when the `enriched_properties` options is used.
    - `after`: The state of the row after the change (`NULL` for deletes).
    - `before`: The state of the row before the change (only present for updates and deletes).
    - `key`: An array composed of the row's `PRIMARY KEY` field(s) (e.g., `[1]` for JSON or `{"id":{"long":1}}` for Avro). The message envelope contains a primary key array when your changefeed is emitting to a sink that does not have a message key as part of its protocol, (e.g., cloud storage, webhook sinks, or Google Pub/Sub). By default, messages emitted to Kafka sinks do not have the primary key array, because the key is part of the message metadata. If you would like messages emitted to Kafka sinks to contain a primary key array, you can use the [`key_in_value`]({% link {{ page.version.version }}/create-changefeed.md %}#key-in-value) option.
    - `op`: The type of change operation.
    - `source`: Cluster, node, and sink information for the data's origin.
    - `ts_ns`: Timestamp of the change event in nanoseconds since the epoch.
- `schema`: The schema and type of each payload field.

Depending on the envelope and options used, changefeed messages can include a variety of fields. Following is a list of the key fields that can appear in changefeed message envelopes, and what each represents:

- `key` – An array of the primary key value(s) for the row that changed. For example, if a table’s primary key is a single column `id = 5`, the key might be `[5]`. For a composite primary key, e.g., `(order_id, line_number)`, the key could look like `[101, 5]`. In Kafka sinks, the `key` is typically delivered as the Kafka message key (not in the JSON payload) by default, but it can be included in the value with certain options (e.g., `key_in_value` for Kafka, or by using a sink like cloud storage where key is part of the JSON). This identifies *which row* the change pertains to.

- `after` – The state of the row **after** the change. This is a JSON object containing column names and values after an insert or update. For delete operations, `after` will be `null` (since after the deletion the row does not exist). In the default `wrapped` envelope, every message for an insert/update has an `"after"` field with the new data. In a `row` envelope, the whole message is effectively the “after” state without a wrapper, and in `key_only` envelope there is no `after` field at all (only the key is sent).

- `before` – The state of the row **before** the change. This field appears only if the `diff` option is enabled on a `wrapped` (or `enriched`) changefeed. For updates, `"before"` is the previous values of the row (prior to the update); for deletes, `"before"` is the last state of the row (since the row is about to be deleted). For inserts, `"before"` will be `null` (the row had no prior state). This field is useful for auditing changes or computing differences. (Not applicable to envelopes like `row` or `key_only` which don’t support a before/after structure.)

- `updated` – A timestamp indicating when the change was committed. This field appears if the `updated` option is enabled. It is usually formatted as a string timestamp (for JSON sinks) in UTC, e.g. `"2025-04-07T20:21:00Z"`. The `updated` timestamp corresponds to the transaction commit time for that change. If the changefeed was started with a cursor (at a specific past timestamp), the updated times will align with the MVCC timestamps of each row version. This field can help consumers order events or measure replication lag. (Not available with `key_only` envelope, since no value payload is present.)

- `mvcc_timestamp` – The internal MVCC timestamp of the change, as a string. This is CockroachDB’s internal timestamp for the version (a decimal number representing logical time). It’s included if the `mvcc_timestamp` option is set. Unlike `"updated"`, which might be omitted for initial historical scans, `"mvcc_timestamp"` is always present on each message when enabled, including backfill events. This field is mainly used for low-level debugging or when exact internal timestamps are needed.

- `source` – Metadata about the **source of the change** event. This is included when using `envelope=enriched` with `enriched_properties='source'` (or `'source,schema'`). The `source` field is a JSON object that can include details such as:
  - `cluster_id` – A unique identifier for the CockroachDB cluster from which the change originated.
  - `node_id` – The ID of the node (within the cluster) that produced the change.
  - `database` / `schema` / `table` – The identifiers of the table that changed (e.g., `online_retail`, `public`, `orders`). This provides the fully qualified table name.
  - Other info as available (e.g., region or tenant name, if applicable in a multi-tenant cluster – not explicitly shown above, but conceptually possible).

- `schema` – Metadata about the **schema** related to the change event. Included with `envelope=enriched` when `enriched_properties='schema'` is used. The `schema` field provides information needed to interpret the data, such as:
  - `table_name` – The name of the table (e.g., `"orders"`).
  - `primary_keys` – An array of primary key column names for that table (e.g., `["id"]` for a single primary key).
  - `columns` – A map or list of column names with their data types (for example, it might list `"id": "INT8", "customer_id": "INT8", "total": "DECIMAL", "status": "STRING"` as in the example above).
  - `version` or schema identifier – In formats like Avro, there might be a schema ID or version, but for JSON this would just directly describe the columns.

- `table` – The name of the table that generated the change. This field appears in certain contexts:
  - In **sinkless changefeeds** (those created with `EXPERIMENTAL CHANGEFEED FOR` or `CREATE CHANGEFEED ... WITH sinkless`), the output includes a `"table"` field for each row change, since a single sinkless feed could cover multiple tables.
  - In the enriched envelope, the fully qualified table name is typically part of the `source` field (as `database`, `schema`, `table` sub-fields), rather than a separate top-level `"table"` field.

- `__crdb__` – A special field used by the `bare` envelope to carry CockroachDB metadata that would otherwise be top-level. When `envelope=bare`, the JSON message’s top level is the row data, so CockroachDB inserts any needed metadata (like the primary key, topic, timestamps, etc.) under a nested `"__crdb__"` object. For example:

    ~~~json
    {
    "__crdb__": {"key": [101]},
    "id": 101,
    "customer_id": 1,
    "total": 50.00,
    "status": "new"
    }
    ~~~

    Here "__crdb__": {"key": [101]} holds the primary key for the row, while the rest of the object are the row’s columns at top level. Other metadata like "updated" timestamps or resolved timestamps would also appear inside __crdb__ if present. This field is specific to the bare envelope or other cases (like custom CDC queries with SELECT) where metadata needs to be attached without interfering with the selected row data.

