# ATTRIBUTES API

Rails 5 has introduced the new **attributes API** which lets model attributes behave in a certain way. It introduces a demarcation in the way the attribute values are stored in the SQL table and accessed outside it.

This eliminates the need of monkey patching and provides a cleaner and better implementation.

The API has been extracted to **ActiveRecord::Attributes::ClassMethods**. It provides two methods:

- `attribute(name, cast_type, **options)`
- `define_attribute(name, cast_type, default: NO_DEFAULT_PROVIDED, user_provided_default: true)`


### `attribute`

- name: the name of the model attribute to be type-casted.
- type: the data-type the attribute is to be type casted in. It is a symbol if it is a primitive data-type, and if you are using custom types, then the type object.
- options:
  - default: gives a default value to the attribute. If not given, use the default value, if provided; else use nil.
  - array: only available for postgresql. Specifies an array type.
  - range: Specifies a range type. Only for postgresql.

## Type casting of attributes

Consider a migration:

```ruby
create_table :todos do |t|
  t.string :name
  t.boolean :completed
  t.decimal :order
end
```

#### In Rails 4

```ruby
class Todo < ActiveRecord::Base
end

t = Todo.new(order: '27.43')
t.order
 => #<BigDecimal:7fa8e431ae90,'0.2743E2',18(27)>
```
#### In Rails 5

```ruby
class Todo < ApplicationRecord
  attribute :order, :integer
end

t = Todo.new(order: '27.43')
t.order
 => 27
 ```

## Default values

It can also be used to give default values.

```ruby
class Todo < ApplicationRecord
  attribute :order, :integer, default: 10
end

t = Todo.new
 => #<Todo id: nil, name: nil, completed: nil, order: 10, created_at: nil, updated_at: nil>
```

## Virtual attributes

It can also be used for virtual attributes - not to be stored in a database table. This feature is only available for Postgres databases.

```ruby
t = Todo.new(va_array: ['1', '2.2', '3'], va_range: '[2,3]')

t.attributes
 => {"id"=>nil, "name"=>nil, "completed"=>nil, "order"=>10, "created_at"=>nil, "updated_at"=>nil, "va_array"=>[1, 2, 3], "va_range"=>2.0..3.0}
```

## Custom attribute types

You can also create your custom types, provided they respond to methods defined on the value type. The method deserialize or cast will be called on your type object, with raw input from the database or from your controllers. See [ActiveRecord::Type::Value](http://edgeapi.rubyonrails.org/classes/ActiveModel/Type/Value.html) for the expected API. It is recommended that your type objects inherit from an existing type, or from ActiveRecord::Type::Value.

You can override the `cast` or `deserialize` methods to define your own custom type. The `deserialize` method takes precedence over the `cast` method. It is called each time your object is instantiated.

```ruby
class CustomType < ActiveRecord::Type::Value
  def cast(value)
    if value =~ /#new_record_prefix/
      value.gsub('#new_record_prefix ', '')
    else
      value
    end
  end
end
```
Now if we create a new record with extra information that does not need to be stored in the database, it will be removed and the record will be stored as needed.

```ruby
Todo.new name: '#new_record_prefix abc'
 => #<Todo id: nil, name: "abc", completed: nil, order: 10, created_at: nil, updated_at: nil>
```

Values passed to ActiveRecord::Base.where can also be customized by overriding the `serialize` method.

```ruby
class CustomType < ActiveRecord::Type::Value
  def serialize(value)
    if value =~ /#where_prefix/
      value.gsub('#where_prefix ', '')
    else
      value
    end
  end
end
```

Now if there's any demarcation in the query string and the value stored in the database table, we can easily get the value.

```ruby
Todo.where name: '#where_prefix abc'
  Todo Load (0.3ms)  SELECT "todos".* FROM "todos" WHERE "todos"."name" = $1  [["name", "abc"]]
 => #<ActiveRecord::Relation [#<Todo id: 25, name: "abc", completed: nil, order: 10, created_at: "2016-05-18 13:22:44", updated_at: "2016-05-18 13:22:44">]>
```

---

### `define_attribute`

This is the low level API which sits beneath `attribute`. It only accepts type objects, and will do its work immediately instead of waiting for the schema to load. Automatic schema detection and #attribute both call this under the hood. While this method is provided so it can be used by plugin authors, application code should probably use #attribute.

- name: The name of the attribute being defined. Expected to be a String.

- cast_type: The type object to use for this attribute.

- default: The default value to use when no value is provided. If this option is not passed, the previous default value (if any) will be used. Otherwise, the default will be nil. A proc can also be passed, and will be called once each time a new value is needed.

- user_provided_default: Whether the default value should be cast using cast or deserialize.
