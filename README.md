# About
ActiveGraph is an object-graph mapping or an OGM. It binds a ruby object to a Neo4J subgraph. You can make changes to this subgraph and persist them. The api tries to be as ActiveRecord compliant as possible to make the transition as simple as possible. The goal is to make an OGM that can be used in production with a Neo4J graph database that provides superior performance. Support for Rails 4.0 and above is in built.

# Usage
ActiveGraph primarily consists of two entities. A node and a relationship. Nodes should represent entities in your domain while relationships connect them. Neo4J gives you a property graph model. This means that both nodes and relationships can have properties. ActiveGraph provides convenient defaults that help you to model your domain in ruby while mainitaing a sync with the schema of your graph. Since a Neo4J instance is not divided into databases, ActiveGraph can be configured to prepend a prefix to all labels created on nodes and relationships with an app name. This name is can be automatically set to your rails app if one is detected. You can turn off this feature or override the app name. ActiveGraph is only a very lightweight OGM wrapper around Neon.

## Configuration
Configuration parameters can be set on `ActiveGraph.config` object or during initialization. To maintain compatibility with `ActiveRecord`, it simply ignores the options it does not use so you don't have to edit your files to make it work. There are two adapters available :-

1. REST
2. Embedded

```ruby
# REST
ActiveGraph.connect(
	adapter: :rest
	host: 'localhost',
    port: '7474',
    prefix: 'MyApp' # prepended to all labels if provided
)

# Embedded
ActiveGraph.connect(
	adapter: :embdedded
    database: 'path to the database'
    prefix: 'MyApp' # prepended to all labels if provided
)

# Alternatively you can provide a block
# Corresponding properties can be set for the embedded adapter
ActiveGraph.initialize do |config|
	config.adapter = :rest
	config.host = 'localhost'
    config.port = 7474,
    config.prefix = 'MyApp'
end
```

This configuration is loaded from the **config/database.yml** in a Rails app.

## Defining the schema
Schemas in Neo4J are optional and therefore ActiveGraph uses the class definition as a schema itself as changing them at any point of time will notbreak your graph or application but only impact the performance due to indexes.

### Nodes
Nodes can be modeled by subclassing `ActiveGraph::Node`. The class name maps to a label with the same name. Unlike ActiveRecord where records are stored as tables which are aggregate data structures and therefore the mapping is pluralized. Therefore in ActiveGraph, class *User* maps to a label named `User` and class *Users* maps to a label named `Users`, both being completely different.
Long story short. A class inheriting from `ActiveGraph::Node` or `ActiveGraph::Relationship` is `ActiveModel` compliant. It has all the goodies you find in ActiveRecord. We'll only lay out the differences here. Most of them are aliases to make the transition very easy but it is recommended you use *ActiveGraph terminology* eventually.

#### Properties
Properties are declared on nodes using the `property` method. You can provide defaults as well as typecast it to a particular class type. An id attribute is automatically available on all entities. `attribute` is an alias. Since there are no migrations or stringent schema, your property declarations define the indexes created on your class label. You can very well use a sublcass without declaring any properties but it is highly recommneded you do.
Neo4J allows you to arbitrarily define properties on nodes and relationships. This is one of the most powerful features of the graph database. There are certain conventions that allow you to define properties at runtime. Access to these properties and relationships is slower than that on defined ones.

```ruby
class User < ActiveGraph::Node
	property :name
end

user = User.new name: "Ujjwal"
user.age = 21 # Defines a property on user at runtime
user.save # Persist the property to the db
yes = user.age? # Returns true
user.age = nil # Deletes the property
no = user.age? # Returns false
user.save # Remvoe the age property from the db
```

#### Relationships
Relationships can be defined on nodes using an ActiveRecord way or an ActiveGraph like everything else.

```ruby
class User < ActiveGraph::Node
	# This creates a relationship named "NATIONAL". We create a relationship when acountry is set and delete it when it is set to nil. Optional properties can be supplied for the relationship
	has_one :country, through: :national, property1, property2
    
    # If you don't care about the type of the relationship then you can omit the through option. It creates a relationshipof type "country".
    has_one :country
    
    # You can supply classes for either of them.
    # Neo4J allows you to have any relationship to any node. Specifying a class creates a constraint on the type of the node.
    has_one Country, through: National
    
	# multiple relationships named "FRIEND" to nodes. Maps the relationship to FriendOf class.
	has_many :users, through: :friend
    
    # This creates an incoming relationship
    belongs_to :organization, through: :employee
    
    # OR
    outgoing :national, to: :country
    
    # Not that the singularity or plurality of the node class names are used to determine wether there is one relationship or many. This forces you to name your nodes and relationships appropriately e.g. naming your node class Friends instead of Friend will lead to a wrong mapping here.
    outgoing :friend, to: :users
    
    incoming :employee, from: :organization
    
    # You can even omit the direction and infer it directly if you know good english
    
    # Create a "NATIONAL" relationship
    national_of :country
    # suffix of to and of create an outgoing relationship
    friend_of :users
    
    # A 'from' suffix creates an incoming relationship. This limits how you name your relationships for better readibility.
    employee_from Organization
    # You can also use the touch option as the last argument to cache results for that particular association.
end
```

Create a relationship to country
```ruby
user = User.new
india = Country.new name: "India"
user.national_of india
```

This deletes the relationship to india and creates one on sweden
```ruby
sweden = Country.new name: "Sweden"
user.national_of sweden
# If you have declared properties on the relationship then you MUST pass a hash of the key-value pairs or they'll be set to arbitrary defaults
user.national_of sweden, property: value
```

You can check if a relationship exists by querying the relationship
```ruby
user.national_of? # Does it exist at all?
user.national_of? india # Does it exist with india
country = user.national.of # Returns the node to which a "national" relationship exists or nil if none or more than one do.
```

You can also create relationships at runtime
```ruby
# Assume User has not relationship declarations.
user = User.new
friend = User.friend
# Single Relationships
# Outgoing
user.knows_to friend
# Incoming
user.knows_from friend
friend == user.friend
# We don't care about the direction
# Defailts to outgoing since our subject is user and not friend. This even reads better
user.knows friend
Access knows properties like this
user.knows.property

# Multiple relationships
another_friend = User.new
user.friends_of friend, another_friend
yet_another_friend = User.new
user.friends << yet_another_friend
user.friends.empty? != user.friends?

# This also creates an employeed_at accessor on the organization object.
user.employeed_at_from organization
same_organization = user.employeed_at
# Assuming organixation has a employees_from schema
yes = organization.employees.include?(user)
yes = user.employed_at?(organization)

# Delete the relationship
user.employed_at nil
```

If you don't follow the conventions and ambiguity arises then we throw a `ActiveGraph::InvalidSchemaError`. Schema created on runtime is not cached and all queries therefore will lead to a database call. This is a penalty on purpose to avoid such usage.

# Architecture
# Notes
It might be worth to try and test the `ActiveRecord` test suite by replacing ActiveRecord with `ActiveGraph` barring the migrations of course.
