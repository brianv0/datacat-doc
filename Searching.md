# Searching Datacat
A typical search will include a path search expression which builds a list of applicable groups and/or folders to search, and a query string.

## Building a List of Search Targets
Currently, search targets are generated using a path, with an optional glob syntax. Internally, Datacat processes the path, starts at the deepest non-glob part of the path, and then starts a walk and collects all matches according to the walk. It's planned to support multiple search targets/globs.

### Path Examples

So, a target path constructed as

`/path/**/folder0[0-5]`

Would match the following results:

* `/path/a/folder01`
* `/path/a/folder05`
* `/path/a/1/folder01`
* `/path/a/1/folder05`
* `/path/b/folder01`
* `/path/b/folder05`
* `/path/b/2/folder01`
* `/path/b/2/folder05`


There are extra flags to restrict searches to groups or folders. For example, a ^ flag would denote only a group, a$ flag would denote only a folder.

Some examples could be this:

* `${path}` - Check **only** the container denoted by path, and return datasets in this containr (implied)
* `${path}/*` - Check datasets in the child containers of this folder
* `${path}/**` - Check any descendent folder or group of this path (recursive)
* `${path}/*^` - Check only the child groups of this path
* `${path}/**^` - Check any descendent group of this path (recursive)
* `${path}/*$` - Check only the child folders of this path
* `${path}/**$` - Check any descendent folder of this path (recursive)

## The Search Langauge

The search language defined, a DSL, is somewhat of a mashup of C, Python, Bash, Ruby/Perl, and SQL expression syntax. This allows users flexibility in how they compose their queries. It only supports boolean expressions and does not understand arithmetic, bitwise, or other expressions.

##### The Basics - Available Syntax

    EQ *            :: "is" | eq(uals?)? | "==" | "="
    NOT_EQ *        :: not\ eq(uals?)? | neq? | "is not" | "!="
    AND             :: "and" | "&amp;&amp;"
    OR              :: "or" | "||"
    LT              :: "lt" | "<"
    LTEQ            :: "lteq" | "le" | "<="
    GT              :: "gt" | ">"
    GTEQ            :: "gteq" | "ge" | ">="

  **Note: **The expanded tokens for EQ and NOT_EQ are as follows:

*   `eq, equal,  equals, is`
*   `not eq, not equal, not equals, ne, neq, is not`

##### Examples
`((metavalue1 == "someString") && (metavalue2 != 1234))`

is equivalent to:

`metavalue1 eq 'someString' and metavalue2 ne 1234`

## Check Lists, Ranges

Check List and Range queries are composed of the search field, and either a list or range of numeric, string, or date literals.

##### Syntax
    IN              :: "in"
    NOT_IN          :: "not in"
    OP_IN           :: IN | NOT_IN
    RANGE           :: ":" | "->" | "to"
    LIST_SPEC<T>    :: T | LIST_SPEC<T> COMMA T
    RANGE_SPEC<T>   :: T RANGE T
    REL_CONTAINS    :: IDENT OP_IN (LIST_SPEC<T> | RANGE_SPEC<T>)
                     | IDENT OP_IN LPAREN (LIST_SPEC<T> | RANGE_SPEC<T>) RPAREN

### Check Lists
Lists provide you with a small collection of values to include in your search. In Python, for example, you might check if a variable, `word` in this case, is in a list:

`word in ('foo','bar','baz')`

If `word` was `foo`, `bar`, or `baz`, This expression would evaluate to true, otherwise it evaluates to false.

In Datacat, you may construct lists the same way, although it is not strictly necessary to include the parentheses.

`metavalue1 in 'hello','world'`

is the same as:

`metavalue1 in ('hello','world')`

One note, however, is that all literals in the list must be of the same type. As a result, the following will throw an exception:

`metavalue1 in ('hello',4)`

You may also negate the check as well:

`metavalue2 not in (1,2,3,5)`

### Ranges
Ranges are a way to check if a value is between, or equal to, two values. That is, ranges are inclusive.

* `metavalue1 in 'hello' to 'world'`
* `metavalue2 in 1->10`
* `metavalue3 in d'2014-02-01':d'2014-03-01'`

Like Check Lists, a Range expression may be surrounded by parenthesis, but it's not mandatory:

`metavalue2 not in (1:10)`

## Matches
Matches are a way of doing wildcard searches and pattern matching on string metadata. Currently, Matches support only two types of wildcards, Single Character wildcards, using the character `?`, and the Kleene Star, `*`, which matches zero or more characters.

Syntax:

    MATCHES         :: "=~" | "matches"
    NOT_MATCHES     :: "!~" | "not matches"

For this example, metavalue1 has these possible values:

`helicopter`, `hello`, `hells`, `help`, `world`

Examples:

`metavalue1 matches 'hell?'`
* result: [`hello`, `hells`]

`metavalue1 =~ 'hel*'`
* result: [`helicopter`,`hello`, `hells`, `help`]

`metavalue1 not matches 'hell?'`
* result: [`helicopter`, `help`, `world`]

`metavalue1 !~ 'world'`
* result: [`helicopter`, `hello`, `hells`, `help`]

**Performance Note:** Matches queries are especially performance sensitive. In general, it's best to use them sparingly and specifically avoid using wildcards, especially the star, at <u>both</u> the beginning and end of the query string **and** `count(results) << count(datasets queried)`.

`metavalue1 =~ '*rl*'`
* result: [`world`]

## Grammar definition

    // Basic Terminals
    IDENT           :: [a-zA-Z_][a-zA-Z0-9_\.\-\:]*
    NUMBER          :: { Python Number Notation }
    STRING          :: { Python 3.0 String Notation }
    BOOLEAN         :: true | false
    NULL            :: null | none
    // Date literal is a python string with 'd' prepended
    DATE            :: 'd' STRING
    
    // Definition of Operators
    NOT_EQ          :: not\ eq(uals?)? | neq? | "is not" | "!="
    AND             :: "and" | "&&"
    OR              :: "or" | "||"
    LT              :: "lt" | "<"
    LTEQ            :: "lteq" | "le" | "<="
    GT              :: "gt" | ">"
    GTEQ            :: "gteq" | "ge" | ">="
    EQ              :: "is" | eq(uals?)? | "==" | "="
    IN              :: "in"
    NOT_IN          :: "not in"
    NOT_MATCHES     :: "!~" | "not matches"
    MATCHES         :: "=~" | "matches"
    
    // Miscellaneous terminals
    COMMA           :: ","
    RANGE           :: ":" | "->" | "to"
    LPAREN          :: "("
    RPAREN          :: ")"
    
    /*--  Parser definitions --*/
    
    // Recursive definition of expression
    EXPR            :: EXPR_OR      | EXPR AND EXPR_OR
    EXPR_OR         :: SIMPLE_EXPR  | EXPR_OR OR SIMPLE_EXPR
    SIMPLE_EXPR     :: REL_EXPR     | LPAREN EXPR RPAREN
    
    // Composite definition of all relations
    REL_EXPR        :: REL_EQUALITY | REL_COMPARISON | REL_MATCH 
                     | REL_RANGE    | REL_CONTAINS
    
    // Definition of Relations
    REL_EQUALITY    :: IDENT OP_EQ ANY_VALUE
    REL_COMPARISON  :: IDENT OP_COMPARISON STRICT_VALUE
    REL_MATCH       :: IDENT MATCHES STRING
                     | IDENT NOT_MATCHES STRING
    REL_CONTAINS    :: IDENT OP_IN (MULTI_SPEC) 
                     | IDENT OP_IN LPAREN (MULTI_SPEC) RPAREN
    
    // Composite definition of operators
    OP_EQ           :: EQ | NOT_EQ
    OP_COMPARISON   :: LT | LTEQ | GT | GTEQ
    OP_IN           :: IN | NOT_IN
    
    // Vector Container Types. T is a homogenous STRICT_VALUE
    MULT_SPEC<T>    :: LIST_SPEC<T> | RANGE_SPEC<T> 
    LIST_SPEC<T>    :: T | LIST_SPEC<T> COMMA T
    RANGE_SPEC<T>   :: T RANGE T

    // Convenience definition of basic terminals
    ANY_VALUE       :: STRICT_VALUE | BOOLEAN | NULL
    STRICT_VALUE    :: NUMBER | STRING | DATE]]>
