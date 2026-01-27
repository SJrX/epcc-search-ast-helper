# EPCC Search AST Helper

## Introduction

This project is designed to help consume the `EP-Internal-Search-Ast-v*` headers. In particular, it provides functions for processing these headers in a variety of use cases.


### Retrieving an AST
The `GetAst()` function will convert the JSON header into a struct that can be then be processed by other functions:

```go
package example

import "github.com/elasticpath/epcc-search-ast-helper"

func Example(headerValue string) (*epsearchast.AstNode, error) {
	
	ast, err := epsearchast.GetAst(headerValue)
	
	if err != nil { 
		return nil, err
    } else { 
		return ast, nil
    }
	
}

```

If the error that comes back is a ValidationErr you should treat it as a 400 to the caller.


### Aliases

This package provides a way to support aliases for fields, this will allow a user to specify multiple different names for a field, and still have it validated and converted properly:

```go
package example

import "github.com/elasticpath/epcc-search-ast-helper"

func Example(ast *epsearchast.AstNode) error {
	
	//The ast from the user will be converted into a new one, and if the user specified a payment_status field, the new ast will have it recorded as status. 
	aliasedAst, err := epsearchast.ApplyAliases(ast, map[string]string{"payment_status": "status"})

	if err != nil { 
		return err
    }
	
	DoSomethingElse(aliasedAst)
	
	return err
}
```

#### Regular Expressions

Aliases can also match Regular Expressions. Regular expresses are specified starting with the `^` and ending with `$`, as the key to the alias. The regular expression can include capture groups and use the same syntax as [Regexp.Expand()](https://pkg.go.dev/regexp#Regexp.Expand) to refer to the groups in the replacement (e.g., `$1`).

**Note**: Regular expressions are an advanced use case. If a field matches more than one key in the configuration (whether exact match, regex, or a combination), validation will return an error listing the conflicting keys. This prevents ambiguous configurations from causing unpredictable behavior.

**Note**: Another catch concerns the fact that `.` is a wild card in regex and often a path separator in JSON, so if you aren't careful you can allow or create inconsistent rules. In general, you should escape `.` in separators to `\.` and use `([^.]+)` to match a wild card part of the attribute name (or maybe even `[a-zA-Z0-9_-]+`) 

**Incorrect**: `^attributes.locales..+.description$` - This would match `attributesXlocalesXXXdescription`, it would also match `attributes.locales.en-US.foo.bar.description`

**Correct**: `^attributes\.locales\.([a-zA-Z0-9_-]+)\.description$`

### Validation

This package provides a concise way to validate that the operators and fields specified in the header are permitted, as well as constrain the allowed values to specific types such as Boolean, Int64, and Float64:

```go
package example

import "github.com/elasticpath/epcc-search-ast-helper"

func Example(ast *epsearchast.AstNode) error {
	var err error
	// The following is an implementation of all the filter operators for orders https://elasticpath.dev/docs/orders/orders-api/orders-api-overview#filtering
	err = epsearchast.ValidateAstFieldAndOperators(ast, map[string][]string {
		"status": {"eq"},
		"payment": {"eq"},
		"shipping": {"eq"},
		"name": {"eq", "like"},
		"email": {"eq", "like"},
		"customer_id": {"eq", "like"},
		"account_id": {"eq", "like"},
		"account_member_id": {"eq", "like"},
		"contact.name": {"eq", "like"},
		"contact.email": {"eq", "like"},
		"shipping_postcode": {"eq", "like"},
		"billing_postcode": {"eq", "like"},
		"with_tax": {"gt", "ge", "lt", "le"},
		"without_tax": {"gt", "ge", "lt", "le"},
		"currency": {"eq"},
		"product_id": {"eq"},
		"product_sku": {"eq"},
		"created_at": {"eq", "gt", "ge", "lt", "le"},
		"updated_at": {"eq", "gt", "ge", "lt", "le"}, 
    })
	
	if err != nil { 
		return err
    }
	
	// You can additionally create aliases which allows for one field to reference another:
	// In this case any headers that search for a field of `order_status` will be mapped to `status` and use those rules instead. 
	err = epsearchast.ValidateAstFieldAndOperatorsWithAliases(ast, map[string][]string {"status": {"eq"}}, map[string]string {"order_status": "status"})
	if err != nil {
		return err
	}
	
	// You can also supply validators on fields, which may be necessary in some cases depending on your data model or to improve user experience.
	// Validation is provided by the go-playground/validator package https://github.com/go-playground/validator#usage-and-documentation
	err = epsearchast.ValidateAstFieldAndOperatorsWithValueValidation(ast, map[string][]string {"status": {"eq"}}, map[string]string {"status": "oneof=incomplete complete processing cancelled"})
	
	if err != nil {
		return err
    }
	
	// Finally you can also restrict certain fields to types, which may be necessary in some cases depending on your data model or to improve user experience.
   err = epsearchast.ValidateAstFieldAndOperatorsWithFieldTypes(ast, map[string][]string {"with_tax": {"eq"}}, map[string]epsearchast.FieldType{"with_tax": epsearchast.Int64})

   if err != nil {
      return err
   }
   
   // All of these options together can be done with  epsearchast.ValidateAstFieldAndOperatorsWithAliasesAndValueValidationAndFieldTypes
	return err
}
```

#### OR Filter Restrictions

By default, when using validation in this library, it will cap the complexity of OR queries to 4. The terminology we use internally is effective index intersection count and conceptually it is computed as follows:

1. The value is 1 for every leaf node in the AST.
2. For AND nodes it is the product of the children.
3. For OR nodes it is the sum of the children.

For example if you were searching for (a=1 OR b=2) AND (c=3 OR d=4 OR e=5), we compute that there might be 6 index intersections needed, (a=1,c=3),(a=1,d=4),(a=1,e=5),... This provides a heuristic to cap costs and prevent 
runaway queries from being generated. It was actually intended that we look at the number of index scans needed, and maybe that's a closer measure to expense in the DB, but the math would only be slightly different.

Over time this value and argument might change as we get more experience, in the interim you can use 0 as a value to allow everything (say if the collection is small).

#### Regular Expressions

Regular Expressions can also be set when using the Validation functions, the same rules apply as for aliases (see above). In general aliases are resolved prior to validation rules and operator checks. If a field matches multiple keys (e.g., both an exact match `array` and a regex `^array.*$`), validation will return an error.

### Working with ASTs

#### Reduce & Semantic Reduce

The library provides two approaches for processing AST trees: `ReduceAst()` and `SemanticReduceAst()`.

##### ReduceAst()

`ReduceAst()` is a low-level generic function that recursively processes an AST tree. It's useful when you need to process all nodes uniformly, regardless of their operator type. For example, extracting all field names, calculating tree depth, or transforming field names.

```go
// Example: Collect all field names from the AST
result, _ := epsearchast.ReduceAst(ast, func(node *epsearchast.AstNode, children []*[]string) (*[]string, error) {
	fields := []string{}
	if len(node.Args) > 0 {
		fields = append(fields, node.Args[0])
	}
	for _, child := range children {
		if child != nil {
			fields = append(fields, *child...)
		}
	}
	return &fields, nil
})
```

##### SemanticReduceAst()

`SemanticReduceAst()` is a higher-level wrapper that uses the `SemanticReducer` interface to provide individual methods for each operator type (VisitEq, VisitLt, etc.). This is the recommended approach for generating queries, as each operator can be translated differently.

```go
// Example: Generate a SQL query using GORM
var qb epsearchast.SemanticReducer[astgorm.SubQuery] = astgorm.DefaultGormQueryBuilder{}
sq, err := epsearchast.SemanticReduceAst(ast, qb)
```

**When to use which:**
- Use `ReduceAst()` when you care about the tree structure but not the specific operators (e.g., collecting field names, calculating depth, transforming field names)
- Use `SemanticReduceAst()` when you need operator-specific behavior (e.g., generating database queries where EQ, LT, GE each translate differently)

#### Customizing ASTs

You can use the `IdentitySemanticReducer` type to simplify rewriting ASTs, by embedding this struct you can only override and process the specific parts you care about. Post-processing the AST tree might be simplier than trying to post process a query written in your langauge, or while rebuilding a query.

#### Util Functions

The library provides several utility functions for working with ASTs:

##### GetAllFirstArgs()/GetAllFirstArgsSorted()/GetAllFirstUnique()

Returns all first arguments (field names) from the AST. Useful for permission checking, index optimization, or field validation.

```go
fields := epsearchast.GetAllFirstArgs(ast)           // []string{"status", "amount", "status"} - includes duplicates
sortedFields := epsearchast.GetAllFirstArgsSorted(ast)  // []string{"amount", "status", "status"} - sorted
uniqueFields := epsearchast.GetAllFirstArgsUnique(ast)  // map[string]struct{}{"status": {}, "amount": {}}
```

##### HasFirstArg()

Returns true if a specific field name appears anywhere in the AST. Useful for quickly checking if a field is referenced before performing expensive operations.

```go
hasStatus := epsearchast.HasFirstArg(ast, "status")  // true if "status" appears as a field name anywhere in the query
```

##### GetAstDepth()

Returns the maximum depth of the AST tree. Useful for limiting query complexity.

```go
depth := epsearchast.GetAstDepth(ast)
```

##### GetEffectiveIndexIntersectionCount()

Returns a heuristic measure of query complexity based on potential index intersections. Used internally to cap OR query complexity (default limit is 4). See the "OR Filter Restrictions" section for more details.

```go
count, err := epsearchast.GetEffectiveIndexIntersectionCount(ast)
```



### Generating Queries

#### GORM/SQL

The following examples shows how to generate a Gorm query with this library.

```go
package example

import "github.com/elasticpath/epcc-search-ast-helper"
import "github.com/elasticpath/epcc-search-ast-helper/gorm"
import "gorm.io/gorm"

func Example(ast *epsearchast.AstNode, query *gorm.DB, tenantBoundaryId string) error {
	var err error
	
	// Not Shown: Validation
	
	// Create query builder
	var qb epsearchast.SemanticReducer[astgorm.SubQuery] = astgorm.DefaultGormQueryBuilder{}

	
	sq, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return err
	}

	// Don't forget to add additional filters 
	query.Where("tenant_boundary_id = ?", tenantBoundaryId)
	
	// Don't forget to expand the Args argument with ...
	query.Where(sq.Clause, sq.Args...)
}
```


##### Limitations

1. The GORM builder does not support aliases (easy MR to fix).
2. The GORM builder does not support joins (fixable in theory).
3. There is no way currently to specify the type of a field for SQL, which means everything gets written as a string today (fixable with MR).
4. The `text` operator implementation makes a number of assumptions, and you likely will want to override its implementation:
   * English is hard coded as the language.
   * Postgres recommends using a [distinct tsvector column and using a stored generated column](https://www.postgresql.org/docs/current/textsearch-tables.html#TEXTSEARCH-TABLES-INDEX). The current implementation does not support this and, you would need to override the method to support it. A simple MR could be made to allow for the Gorm query builder to know if there is a tsvector column and use that.

##### Advanced Customization

In some cases you may want to change the behaviour of the generated SQL, the following example shows how to do that
in this case, we want all eq queries for emails to use the lower case, comparison, and for cart_items field to be numeric.

```go
package example

import (
	"github.com/elasticpath/epcc-search-ast-helper"
	"strconv"
)
import "github.com/elasticpath/epcc-search-ast-helper/gorm"
import "gorm.io/gorm"


func Example(ast *epsearchast.AstNode, query *gorm.DB, tenantBoundaryId string) error {
	var err error

	// Not Shown: Validation

	// Create query builder
	var qb epsearchast.SemanticReducer[astgorm.SubQuery] = &CustomQueryBuilder{}

	sq, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return err
	}

	// Don't forget to add additional filters 
	query.Where("tenant_boundary_id = ?", tenantBoundaryId)
	
	// Don't forget to expand the Args argument with ...
	query.Where(sq.Clause, sq.Args...)
}

type CustomQueryBuilder struct {
	astgorm.DefaultGormQueryBuilder
}

func (l *CustomQueryBuilder) VisitEq(first, second string) (*astgorm.SubQuery, error) {
	if first == "email" {
		return &astgorm.SubQuery{
			Clause: fmt.Sprintf("LOWER(%s::text) = LOWER(?)", first),
			Args:   []interface{}{second},
		}, nil
	} else if first == "cart_items" {
		n, err := strconv.Atoi(second)
		if err != nil {
			return nil, err
		}
		return &astgorm.SubQuery{
			Clause: fmt.Sprintf("%s = ?", first),
			Args:   []interface{}{n},
		}, nil
	} else {
		return DefaultGormQueryBuilder.VisitEq(l.DefaultGormQueryBuilder, first, second)
	}
}
```

#### Mongo

The following examples shows how to generate a Mongo Query with this library.

```go
package example

import (
	"context"
	"github.com/elasticpath/epcc-search-ast-helper"
	"github.com/elasticpath/epcc-search-ast-helper/mongo"
	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
)

func Example(ast *epsearchast.AstNode, collection *mongo.Collection, tenantBoundaryQuery bson.M)  (*mongo.Cursor, error) {
	// Not Shown: Validation

	// Create query builder
	var qb epsearchast.SemanticReducer[bson.D] = DefaultMongoQueryBuilder{}

	// Create Query Object
	queryObj, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return nil, err
	}

	mongoQuery := bson.D{
		{"$and",
			bson.A{
				tenantBoundaryQuery,
				queryObj,
			},
		}}
	
	
	return collection.Find(context.TODO(), mongoQuery)
}
```


##### Limitations

1. The Mongo Query builder is designed to produce filter compatible with the [filter argument in a Query](https://www.mongodb.com/docs/drivers/go/current/fundamentals/crud/read-operations/query-document/#specify-a-query), if a field in the API is a projection that requires computation via the aggregation pipeline, then we would likely need code changes to support that.
2. The [$text](https://www.mongodb.com/docs/v7.0/reference/operator/query/text/#behavior) operator in Mongo has a number of limitations that make it unsuitable for arbitrary queries. In particular in mongo you can only search a collection, not fields for text data, and you must declare a text index. This means that any supplied field in the filter, is just dropped. It is recommended that when using `text` with Mongo, you only allow users to search `text(*,search)` , i.e., force them to use a wildcard as the field name. It is also recommended that you use a [Wildcard](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-text/create-wildcard-text-index/) index to avoid the need of having to remove and modify it over time.

##### Advanced Customization


###### Field Types

In some cases, depending on how data is stored in Mongo you might need to instruct the query builder what the type of the field is. The following example shows how to do that in this case we want to specify that `with_tax` is a number.

```go
package example

import (
	"context"
	"github.com/elasticpath/epcc-search-ast-helper"
	"github.com/elasticpath/epcc-search-ast-helper/mongo"
	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
	"strings"
)

func Example(ast *epsearchast.AstNode, collection *mongo.Collection, tenantBoundaryQuery *bson.M)  (*mongo.Cursor, error) {
	// Not Shown: Validation

	// Create query builder
	var qb epsearchast.SemanticReducer[bson.D] = &astmongo.DefaultMongoQueryBuilder{
		FieldTypes: map[string]astmongo.FieldType{"with_tax": astmongo.Int64},
    }

	// Create Query Object
	queryObj, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return nil, err
	}

	mongoQuery := bson.D{
		{"$and",
			bson.A{
				tenantBoundaryQuery,
				queryObj,
			},
		}}
	
	return collection.Find(context.TODO(), mongoQuery)
}
```

###### Custom Queries

In some cases you may want to change the behaviour of the generated Mongo, the following example shows how to do that in this case we want to change emails because
we store them only in lower case in the db.

```go
package example

import (
	"context"
	"github.com/elasticpath/epcc-search-ast-helper"
	"github.com/elasticpath/epcc-search-ast-helper/mongo"
	"go.mongodb.org/mongo-driver/v2/bson"
	"go.mongodb.org/mongo-driver/v2/mongo"
	"strings"
)

func Example(ast *epsearchast.AstNode, collection *mongo.Collection, tenantBoundaryQuery *bson.M)  (*mongo.Cursor, error) {
	// Not Shown: Validation

	// Create query builder
	var qb epsearchast.SemanticReducer[bson.D] = &LowerCaseEmail{}

	// Create Query Object
	queryObj, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return nil, err
	}

	mongoQuery := bson.D{
		{"$and",
			bson.A{
				tenantBoundaryQuery,
				queryObj,
			},
		}}
	
	return collection.Find(context.TODO(), mongoQuery)
}

type LowerCaseEmail struct {
	astmongo.DefaultMongoQueryBuilder
}

func (l *LowerCaseEmail) VisitEq(first, second string) (*bson.D, error) {
	if first == "email" {
		return &bson.D{{first, bson.D{{"$eq", strings.ToLower(second)}}}}, nil
	} else {
		return DefaultMongoQueryBuilder.VisitEq(l.DefaultMongoQueryBuilder, first, second)
	}
}
```

You can of course use the `FieldTypes` and `CustomQueryBuilder` together.

#### Elasticsearch (Open Search)

The following examples shows how to generate an Elasticsearch Query with this library.

```go
package example
import "github.com/elasticpath/epcc-search-ast-helper"
import "github.com/elasticpath/epcc-search-ast-helper/els"


var qb = &LowerCaseEmail{
   astes.DefaultEsQueryBuilder{
      OpTypeToFieldNames: map[string]*astes.OperatorTypeToMultiFieldName{
         "status": {
            Wildcard: "status.wildcard",
         },
      },
   },
}

func init() {
	// Check all the options are valid.
	// Doing this in an init method, ensures that you don't have issues at runtime.
    qb.MustValidate()	
}

func Example(ast *epsearchast.AstNode, tenantBoundaryId string)  (string, error) {
   // Not Shown: Validation
	

   // Create Query Object
   query, err := astes.SemanticReduceAst[astes.JsonObject](astNode, qb)

   if err != nil {
      return nil, err
   }
   
   // Verification
   queryJson, err := json.MarshalIndent(query, "", "  ")

}

type LowerCaseEmail struct {
   astes.DefaultEsQueryBuilder
}

func (l *LowerCaseEmail) VisitEq(first, second string) (*astes.JsonObject, error) {
   if first == "email" {
      return astes.DefaultEsQueryBuilder.VisitEq(l.DefaultEsQueryBuilder, first, strings.ToLower(second))
   } else {
      return astes.DefaultEsQueryBuilder.VisitEq(l.DefaultEsQueryBuilder, first, second)
   }
}

```

##### Limitations

1. There is no support for [Null Values](https://opensearch.org/docs/latest/field-types/supported-field-types/index/#null-value), so while the is_null key is supported it defaults to empty
   * An MR would be welcome to fix this.
2. Elastic/OpenSearch do not by default ensure that objects retain their relations (e.g, you can't search for nested subobjects that have the AND of two properties). In order to support this you need to use [Nested Objects](https://opensearch.org/docs/latest/field-types/supported-field-types/nested/).
3. You cannot use the is_null operator with nested fields.
   * It's unclear whether or not this could actually be supported nicely.

##### Advanced Customization

###### Field Types

Elasticsearch may store the same field in multiple ways using [multi-fields](https://opensearch.org/docs/latest/field-types/supported-field-types/index/#multifields), and depending on the operator being used you might need to use a different field (e.g., `text(a,"hello")` could use a `text` field called `a`, but `eq(a,"hello")` might need the `keyword` field `a.keyword`).
You can use the OpTypeToFieldNames map to essentially change the field to look at based on the operator type, check the code but there are essentially a number of classes, such as equality, relational, text, array, and wildcard. 

###### Nested Subqueries

Elasticsearch has a number of limitations when storing data to be mindful of:

1. It doesn't [natively support arrays](https://www.elastic.co/guide/en/elasticsearch/reference/current/array.html). Instead, multiple elements in a field are treated as a [Set](https://en.wikipedia.org/wiki/Set_(mathematics). Concretely this makes it difficult to support filters such as `eq(parent[0],foo)`, as ES is only really designed to support queries such as `contains(parent,foo)`. 
   * You can't use dynamic field names to get around this, as there is an [upper limit of the number of fields that can be used](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-settings-limit.html) within an index.
2. Elasticsearch has [no concept of inner objects](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html#nested-arrays-flattening-objects), so if your primary storage engine is a document store the association between objects distinct fields is lost. From their documentation, if a document has the structure `<users: [<first: John, last: Smith>, <first: Alice, last: White>]>`, Elastic Search persists `<users.first: {Alice, John}, users.last: {Smith, White}>`. Elasticsearch can't distinguish between "John Smith", "Alice White" and "John White" and "Alice Smith".

This makes it challenging to support filters such as `eq(parent[0],foo)` or `text(locale.FR.description,"touté")` natively. In order to support these kinds of searches, the way this library currently supports is to use the [nested field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html) to store the data. Conceptually whereas another database might store the data as `<parent: [foo,bar]>`, we can store the data as: `<parent:{<idx:0, value:foo>,<idx:1, value:bar>}>`, this means that conceptually the library would translate `eq(parent[0],foo)` to something like `eq(parent.idx,0):eq(parent.value,foo)`, and then wrap the resulting query in a [nested query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html). 

This library includes support for automatically creating these nested fields provided that you have an index element on each field. Please see the integration tests, for examples of how to use this feature.

###### Overriding Behaviour

The Elasticsearch Query Builder has a couple of family of methods that can be overridden:

1. `Visit___()` - These functions override what happens when we see particular nodes in the AST. These functions return the resulting JSON to query Elasticsearch with, and do so by generating a builder, and then handing it off to the nested query logic to decode the field name, etc...
2. `Get_____QueryBuilder()` - These functions override the resulting ES queries that are built. These functions return a function that returns the JSON to query Elasticsearch With.

In Mongo and Postgres there is a near 1-1 translation between an AST node and a query. In Elasticsearch, due to [Nested Queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-nested-query.html) the mapping is not 1-to-1,
due to visiting a nested field. If you need to override behaviour pertaining to a nested field, the `Get____QueryBuilder()` functions are probably where the override should happen, otherwise `Visit____()` might be simpler.

#### MongoDB Atlas Search (Beta)

The following example shows how to generate a MongoDB Atlas Search query with this library.

**Note**: MongoDB Atlas Search support is currently in beta. Some operators & types are not yet implemented.

```go
package example

import (
	"context"
	"github.com/elasticpath/epcc-search-ast-helper"
	"github.com/elasticpath/epcc-search-ast-helper/mongo"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
)

func Example(ast *epsearchast.AstNode, collection *mongo.Collection, tenantBoundaryId string) (*mongo.Cursor, error) {
	// Not Shown: Validation

	// Create Atlas Search query builder
	// Configure multi-analyzers for fields that support LIKE/ILIKE
	var qb epsearchast.SemanticReducer[bson.D] = astmongo.DefaultAtlasSearchQueryBuilder{
		FieldToMultiAnalyzers: map[string]*astmongo.StringMultiAnalyzers{
			"name": {
				WildcardCaseInsensitive: "caseInsensitiveAnalyzer",
				WildcardCaseSensitive:   "caseSensitiveAnalyzer",
			},
			"email": {
				WildcardCaseInsensitive: "caseInsensitiveAnalyzer",
				WildcardCaseSensitive:   "caseSensitiveAnalyzer",
			},
		},
	}

	// Create AST Query Object
	astQuery, err := epsearchast.SemanticReduceAst(ast, qb)

	if err != nil {
		return nil, err
	}

	// Build the Atlas Search query with compound must clause
	// - astQuery contains the user's search filter (from AST)
	// - equals clauses ensure results are scoped to the tenant boundary
	searchQuery := bson.D{
		{"compound",
			bson.D{
				{"must", bson.A{
					astQuery,
					bson.D{
						{"equals", bson.D{
							{"path", "tenant_boundary_id"},
							{"value", tenantBoundaryId},
						}},
					},
				}},
			},
		},
	}

	// Execute the search using aggregation pipeline
	pipeline := mongo.Pipeline{
		{{Key: "$search", Value: searchQuery}},
	}

	return collection.Aggregate(context.TODO(), pipeline)
}
```

##### Supported Operators

The following operators are currently supported:
- `text` - Full-text search with analyzers
- `eq` - Exact case-sensitive equality matching (string fields only)
- `in` - Multiple value exact matching (string fields only)
- `like` - Case-sensitive wildcard matching
- `ilike` - Case-insensitive wildcard matching
- `gt` - Greater than (lexicographic comparison for strings)
- `ge` - Greater than or equal (lexicographic comparison for strings)
- `lt` - Less than (lexicographic comparison for strings)
- `le` - Less than or equal (lexicographic comparison for strings)

##### Field Configuration

###### Multi-Analyzer Configuration for LIKE/ILIKE

To support `like` and `ilike` operators with proper case sensitivity handling, you need to:

1. **Define custom analyzers in your search index** with appropriate tokenization and case handling
2. **Configure multi-analyzers on your string fields** to index the same field with different analyzers
3. **Map fields to analyzer names** in the query builder using `FieldToMultiAnalyzers`

**Example Search Index Definition:**

```json
{
  "analyzers": [
    {
      "name": "caseInsensitiveAnalyzer",
      "tokenizer": {
        "type": "keyword"
      },
      "tokenFilters": [
        {
          "type": "lowercase"
        }
      ]
    }
  ],
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        {
          "type": "string",
          "analyzer": "lucene.standard",
          "multi": {
            "caseInsensitiveAnalyzer": {
              "type": "string",
              "analyzer": "caseInsensitiveAnalyzer"
            },
            "caseSensitiveAnalyzer": {
              "type": "string",
              "analyzer": "lucene.keyword"
            }
          }
        },
        {
          "type": "token"
        }
      ]
    }
  }
}
```

**Query Builder Configuration:**

The `FieldToMultiAnalyzers` map specifies which multi-analyzer to use for each field:

```go
FieldToMultiAnalyzers: map[string]*StringMultiAnalyzers{
	"name": {
		WildcardCaseInsensitive: "caseInsensitiveAnalyzer",      // Used for ILIKE
		WildcardCaseSensitive:   "caseSensitiveAnalyzer",  // Used for LIKE
	},
}
```

**Behavior:**
If a field is **not** in `FieldToMultiAnalyzers`, if you specify a non empty analyzer, then a "multi" attribute is generated with the name (e.g., `{"path": {"value": "fieldName", "multi": "analyzerName"}}`)

This allows you to mix fields with and without multi-analyzer support in the same index.

##### Limitations

1. The following operators are not yet implemented: `contains`, `contains_any`, `contains_all`, `is_null`
2. The following field types are not currently supported: UUID fields, Date fields, Numeric fields (numbers are compared as strings)
3. Range operators (`gt`, `ge`, `lt`, `le`) perform lexicographic comparison on string fields only
4. Atlas Search requires proper [search index configuration](https://www.mongodb.com/docs/atlas/atlas-search/create-index/) with appropriate field types:
   - String fields used with `like`/`ilike` should be indexed with multi-analyzers as shown above
   - String fields used with `eq`/`in` should be indexed with `token` type
   - String fields used with range operators (`gt`/`ge`/`lt`/`le`) work with `token` type for lexicographic comparison
   - Text fields should be indexed with `string` type and an appropriate analyzer
5. Unlike regular MongoDB queries, Atlas Search queries use the aggregation pipeline with the `$search` stage
6. Additional filters (like tenant boundaries) should be included within the `$search` stage using compound must clauses for optimal performance (as shown in the example above). Alternatively, they can be added as separate `$match` stages after the `$search` stage, though this is less efficient as it filters results after the search rather than during indexing

### FAQ

#### Design

##### Why does validation include alias resolution, why not process aliases first?

When validation errors occur, those errors go back to the user, so telling the user the error that occurred using the term they specified improves usability.

##### Why does the ES only support nested fields, and not other techniques such as flattened or object.

Nested queries are the most *powerful* and *flexible* ways from a user perspective, however they are likely also the slowest, and eat up document ids a lot. In the future as other operation concerns become an issue, support can be added.