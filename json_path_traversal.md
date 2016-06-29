In BigQuery, there's sufficient metadata for Looker to generate a dimension containing the path for a nested object. Consider the following:
```json
 "event_type" : "run_query",
 "event_attributes" : 
 	{ "query" : 
 		{
	 		"fields" : ["foo", "bar", "baz"],
	 		"limit" : 500
 		}
	}
}
```
Provided we've taught BigQuery how to traverse this object, it'd generate the following:
```lookml
- dimension: event_attributes_query_limit
  type: number
  sql: ${TABLE}.event_attributes.query.limit
```
Super fantastic!

For non-BigQuery users, they're somewhat hosed. They need prior knowledge of all their sweet properties in their JSON in order to create dimensions. What if Looker just figured it out for them?

Part of the generator could traverse a JSON or HStore column, find all of the unique properties, their corresponding paths and data types. Then it could generate dialect-specific dimensions with path traversal.

Consider an example of sparse data (`my_doc.json`):
```json
{"id" : 1,
 "event_type" : "run_query",
 "event_attributes" : 
 	{ "query" : 
 		{
	 		"fields" : ["foo", "bar", "baz"],
	 		"limit" : 500
 		}
	}
}
{"id" : 2,
 "event_type" : "save_look",
 "event_attributes" : 
 	{ "type" : "column",
 		"config" : {
	 		"series_labels" : ["slick dimension", "bitchin measure"],
	 		"table_theme" : "classic"
 		}
	}
}
```

```ruby
require 'json'

def traverse(object, path = '', divider = '.', &block)
  if object.is_a?(Hash)
    object.each do |key, value|
      traverse(value, "#{path}#{path.empty? ? '' : divider}#{key}", divider, &block)
    end
  else
    yield path, object
  end
end

def lookml_type(type)
  case
  when (type.is_a?(Float) || type.is_a?(Integer))
    then 'number'
  when (type.is_a?(TrueClass) || type.is_a?(FalseClass))
    then 'yesno'
  else 'string'
  end
end

def generate_lookml(name, type, sql)
  "- dimension: #{name}\n\s\stype: #{type}\n\s\ssql: ${TABLE}.#{sql}\n"
end

paths = {}
File.open("my_doc.json").each do |record|
  parsed_record = JSON.parse(record)
  traverse(parsed_record) { |path, value| paths.merge!(path => lookml_type(value)) }
end

paths.each do |k, v|
  name = k.include?(".") ? k.gsub(".", "_") : k
  generate_lookml(name, v, k)
end
```

Returns

```
- dimension: id
  type: number
  sql: ${TABLE}.id

- dimension: event_type
  type: string
  sql: ${TABLE}.event_type

- dimension: event_attributes_query_fields
  type: string
  sql: ${TABLE}.event_attributes.query.fields

- dimension: event_attributes_query_limit
  type: number
  sql: ${TABLE}.event_attributes.query.limit

- dimension: event_attributes_type
  type: string
  sql: ${TABLE}.event_attributes.type

- dimension: event_attributes_config_series_labels
  type: string
  sql: ${TABLE}.event_attributes.config.series_labels

- dimension: event_attributes_config_table_theme
  type: string
  sql: ${TABLE}.event_attributes.config.table_theme
```

In this example, I'm using `.` as the scoping operator. For PostgreSQL or HStore, we could replace it with `->`, _e.g._, 
```lookml
- dimension: event_attributes_config_table_theme
  type: string
  sql: ${TABLE}.event_attributes -> config -> table_theme
```
