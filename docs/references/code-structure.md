# Code Structure

## pkg/templates

### Template

Template is the basic unit of input to the engine which describes the requests to be made, matching to be done, data to extract, etc.

The template structure is described here. Template level attributes are defined here as well as convenience methods to validate, parse and compile templates creating executers. 

Any attributes etc. required for the template, engine or requests to function are also set here.

Workflows are also compiled, their templates are loaded and compiled as well. Any validations etc. on the paths provided are also done here.

`Parse` function is the main entry point which returns a template for a `filePath` and `executorOptions`. It compiles all the requests for the templates, all the workflows, as well as any self-contained request etc. It also caches the templates in an in-memory cache.

### Preprocessors

Preprocessors are also applied here which can do things at template level. They get data of the template which they can alter at will on runtime. This is used in the engine to do random string generation.

Custom processor can be used if they satisfy the following interface.

```go
type Preprocessor interface {
	Process(data []byte) []byte
}
```

## pkg/model

Model package implements Information structure for Nuclei Templates. `Info` contains all major metadata information for the template. `Classification` structure can also be used to provide additional context to vulnerability data.

It also specifies a `WorkflowLoader` interface that is used during workflow loading in template compilation stage.

```go
type WorkflowLoader interface {
	GetTemplatePathsByTags(tags []string) []string
	GetTemplatePaths(templatesList []string, noValidate bool) []string
}
```

## pkg/protocols

Protocols package implements all the request protocols supported by Nuclei. This includes http, dns, network, headless and file requests as of now. 

### Request

It exposes a `Request` interface that is implemented by all the request protocols supported.

```go
// Request is an interface implemented any protocol based request generator.
type Request interface {
	Compile(options *ExecuterOptions) error
	Requests() int
	GetID() string
	Match(data map[string]interface{}, matcher *matchers.Matcher) (bool, []string)
	Extract(data map[string]interface{}, matcher *extractors.Extractor) map[string]struct{}
	ExecuteWithResults(input string, dynamicValues, previous output.InternalEvent, callback OutputEventCallback) error
	MakeResultEventItem(wrapped *output.InternalWrappedEvent) *output.ResultEvent
	MakeResultEvent(wrapped *output.InternalWrappedEvent) []*output.ResultEvent
	GetCompiledOperators() []*operators.Operators
}
```

Many of these methods are similar across protocols while some are very protocol specific. 

A brief overview of the methods is provided below -

- **Compile** - Compiles the request with provided options.
- **Requests** - Returns total requests made.
- **GetID** - Returns any ID for the request
- **Match** - Used to perform matching for patterns using matchers
- **Extract** - Used to perform extraction for patterns using extractors
- **ExecuteWithResults** - Request execution function for input.
- **MakeResultEventItem** - Creates a single result event for the intermediate `InternalWrappedEvent` output structure.
- **MakeResultEvent** - Returns a slice of results based on an `InternalWrappedEvent` internal output event.
- **GetCompiledOperators** - Returns the compiled operators for the request.

`MakeDefaultResultEvent` function can be used as a default for `MakeResultEvent` function when no protocol-specific features need to be implemented for result generation. 

For reference protocol requests implementations, one can look at the below packages  - 

1. [pkg/protocols/http](./v2/pkg/protocols/http)
2. [pkg/protocols/dns](./v2/pkg/protocols/dns)
3. [pkg/protocols/network](./v2/pkg/protocols/network)

### Executer

All these different requests interfaces are converted to an Executer which is also an interface defined in `pkg/protocols` which is used during final execution of the template.

```go
// Executer is an interface implemented any protocol based request executer.
type Executer interface {
	Compile() error
	Requests() int
	Execute(input string) (bool, error)
	ExecuteWithResults(input string, callback OutputEventCallback) error
}
```

The `ExecuteWithResults` function accepts a callback, which gets provided with results during execution in form of `*output.InternalWrappedEvent` structure.

The default executer is provided in `pkg/protocols/common/executer` . It takes a list of Requests and relevant `ExecuterOptions` and implements the Executer interface required for template execution. The executer during Template compilation process is created from this package and used as-is.

A different executer is the Clustered Requests executer which implements the Nuclei Request clustering functionality in `pkg/templates`  We have a single HTTP  request in cases where multiple templates can be clustered and multiple operator lists to match/extract. The first HTTP request is executed while all the template matcher/extractor are evaluated separately.

For Workflow execution, a separate RunWorkflow function is used which executes the workflow independently of the template execution.

With this basic premise set, we can now start exploring the current runner implementation which will also walk us through the architecture of nuclei.

## internal/runner

### Template loading

The first process after all CLI specific initialisation is the loading of template/workflow paths that the user wants to run. This is done by the packages described below.

#### pkg/catalog

This package is used to get paths using mixed syntax. It takes a template directory and performs resolving for template paths both from provided template and current user directory.

The syntax is very versatile and can include filenames, glob patterns, directories, absolute paths, and relative-paths.



Next step is the initialisation of the reporting modules which is handled in `pkg/reporting`.  

#### pkg/reporting

Reporting module contains exporters and trackers as well as a module for deduplication and a module for result formatting. 

Exporters and Trackers are interfaces defined in pkg/reporting.

```go
// Tracker is an interface implemented by an issue tracker
type Tracker interface {
	CreateIssue(event *output.ResultEvent) error
}

// Exporter is an interface implemented by an issue exporter
type Exporter interface {
	Close() error
	Export(event *output.ResultEvent) error
}
```

Exporters include `Elasticsearch`, `markdown`, `sarif` . Trackers include `GitHub` , `Gitlab` and `Jira`.

Each exporter and trackers implement their own configuration in YAML format and are very modular in nature, so adding new ones is easy.



After reading all the inputs from various sources and initialisation other miscellaneous options, the next bit is the output writing which is done using `pkg/output` module.

#### pkg/output

Output package implements the output writing functionality for Nuclei.

Output Writer implements the Writer interface which is called each time a result is found for nuclei.

```go
// Writer is an interface which writes output to somewhere for nuclei events.
type Writer interface {
	Close()
	Colorizer() aurora.Aurora
	Write(*ResultEvent) error
	Request(templateID, url, requestType string, err error)
}
```

ResultEvent structure is passed to the Nuclei Output Writer which contains the entire detail of a found result. Various intermediary types like `InternalWrappedEvent` and `InternalEvent` are used throughout nuclei protocols and matchers to describe results in various stages of execution.

Interactsh is also initialised if it is not explicitly disabled. 

#### pkg/protocols/common/interactsh

Interactsh module is used to provide automatic Out-of-Band vulnerability identification in Nuclei. 

It uses two LRU caches, one for storing interactions for request URLs and one for storing requests for interaction URL. These both caches are used to correlated requests received to the Interactsh OOB server and Nuclei Instance. [Interactsh Client](https://github.com/projectdiscovery/interactsh/pkg/client) package does most of the heavy lifting of this module.

Polling for interactions and server registration only starts when a template uses the interactsh module and is executed by nuclei. After that no registration is required for the entire run.



### RunEnumeration

Next we arrive in the `RunEnumeration` function of the runner.

`HostErrorsCache` is initialised which is used throughout the run of Nuclei enumeration to keep track of errors per host and skip further requests if the errors are greater than the provided threshold. The functionality for the error tracking cache is defined in [hosterrorscache.go](https://github.com/projectdiscovery/nuclei/blob/master/v2/pkg/protocols/common/hosterrorscache/hosterrorscache.go) and is pretty simplistic in nature.

Next the `WorkflowLoader` is initialised which used to load workflows. It exists in `v2/pkg/parsers/workflow_loader.go`

The loader is initialised moving forward which is responsible for Using Catalog, Passed Tags, Filters, Paths, etc. to return compiled `Templates` and `Workflows`. 

#### pkg/catalog/loader

First the input passed by the user as paths is normalised to absolute paths which is done by the `pkg/catalog` module.  Next the path filter module is used to remove the excluded template/workflows paths.

`pkg/parsers` module's `LoadTemplate`,`LoadWorkflow` functions are used to check if the templates pass the validation + are not excluded via tags/severity/etc. filters. If all checks are passed, then the template/workflow is parsed and returned in a compiled form by the `pkg/templates`'s `Parse` function.

`Parse` function performs compilation of all the requests in a template + creates Executers from them returning a runnable Template/Workflow structure.

Clustering module comes in next whose job is to cluster identical HTTP GET requests together (as a lot of the templates perform the same get requests many times, it's a good way to save many requests on large scans with lots of templates). 

### pkg/operators

Operators package implements all the matching and extracting logic of Nuclei. 

```go
// Operators contain the operators that can be applied on protocols
type Operators struct {
	Matchers []*matchers.Matcher
	Extractors []*extractors.Extractor
	MatchersCondition string
}
```

A protocol only needs to embed the `operators.Operators` type shown above, and it can utilise all the matching/extracting functionality of nuclei.

```go
// MatchFunc performs matching operation for a matcher on model and returns true or false.
type MatchFunc func(data map[string]interface{}, matcher *matchers.Matcher) (bool, []string)

// ExtractFunc performs extracting operation for an extractor on model and returns true or false.
type ExtractFunc func(data map[string]interface{}, matcher *extractors.Extractor) map[string]struct{}

// Execute executes the operators on data and returns a result structure
func (operators *Operators) Execute(data map[string]interface{}, match MatchFunc, extract ExtractFunc, isDebug bool) (*Result, bool) 
```

The core of this process is the Execute function which takes an input dictionary as well as a Match and Extract function and return a `Result` structure which is used later during nuclei execution to check for results.

```go
// Result is a result structure created from operators running on data.
type Result struct {
	Matched bool
	Extracted bool
	Matches map[string][]string
	Extracts map[string][]string
	OutputExtracts []string
	DynamicValues map[string]interface{}
	PayloadValues map[string]interface{}
}
```

The internal logics for matching and extracting for things like words, regexes, jq, paths, etc. is specified in `pkg/operators/matchers`, `pkg/operators/extractors`. Those packages should be investigated for further look into the topic.


### Template Execution

`pkg/core` provides the engine mechanism which runs the templates/workflows on inputs. It exposes an `Execute` function which does the task of execution while also doing template clustering. The clustering can also be disabled optionally by the user.
 
An example of using the core engine is provided below.

```go
engine := core.New(r.options)
engine.SetExecuterOptions(executerOpts)
results := engine.ExecuteWithOpts(finalTemplates, r.hmapInputProvider, true)
```

## Project Structure

- [v2/pkg/reporting](./v2/pkg/reporting) - Reporting modules for nuclei.
- [v2/pkg/reporting/exporters/sarif](./v2/pkg/reporting/exporters/sarif) - Sarif Result Exporter
- [v2/pkg/reporting/exporters/markdown](./v2/pkg/reporting/exporters/markdown) - Markdown Result Exporter
- [v2/pkg/reporting/exporters/es](./v2/pkg/reporting/exporters/es) - Elasticsearch Result Exporter
- [v2/pkg/reporting/dedupe](./v2/pkg/reporting/dedupe) - Dedupe module for Results
- [v2/pkg/reporting/trackers/gitlab](./v2/pkg/reporting/trackers/gitlab) - Gitlab Issue Tracker Exporter
- [v2/pkg/reporting/trackers/jira](./v2/pkg/reporting/trackers/jira) - Jira Issue Tracker Exporter
- [v2/pkg/reporting/trackers/github](./v2/pkg/reporting/trackers/github) - GitHub Issue Tracker Exporter
- [v2/pkg/reporting/format](./v2/pkg/reporting/format) - Result Formatting Functions
- [v2/pkg/parsers](./v2/pkg/parsers) - Implements template as well as workflow loader for initial template discovery, validation and - loading.
- [v2/pkg/types](./v2/pkg/types) - Contains CLI options as well as misc helper functions.
- [v2/pkg/progress](./v2/pkg/progress) - Progress tracking
- [v2/pkg/operators](./v2/pkg/operators) - Operators for Nuclei
- [v2/pkg/operators/common/dsl](./v2/pkg/operators/common/dsl) - DSL functions for Nuclei YAML Syntax
- [v2/pkg/operators/matchers](./v2/pkg/operators/matchers) - Matchers implementation
- [v2/pkg/operators/extractors](./v2/pkg/operators/extractors) - Extractors implementation
- [v2/pkg/catalog](./v2/pkg/catalog) - Template loading from disk helpers
- [v2/pkg/catalog/config](./v2/pkg/catalog/config) - Internal configuration management
- [v2/pkg/catalog/loader](./v2/pkg/catalog/loader) - Implements loading and validation of templates and workflows.
- [v2/pkg/catalog/loader/filter](./v2/pkg/catalog/loader/filter) - Filter filters templates based on tags and paths
- [v2/pkg/output](./v2/pkg/output) - Output module for nuclei
- [v2/pkg/workflows](./v2/pkg/workflows) - Workflow execution logic + declarations
- [v2/pkg/utils](./v2/pkg/utils) - Utility functions
- [v2/pkg/model](./v2/pkg/model) - Template Info + misc
- [v2/pkg/templates](./v2/pkg/templates) - Templates core starting point
- [v2/pkg/templates/cache](./v2/pkg/templates/cache) - Templates cache
- [v2/pkg/protocols](./v2/pkg/protocols) - Protocol Specification
- [v2/pkg/protocols/file](./v2/pkg/protocols/file) - File protocol
- [v2/pkg/protocols/network](./v2/pkg/protocols/network) - Network protocol
- [v2/pkg/protocols/common/expressions](./v2/pkg/protocols/common/expressions) - Expression evaluation + Templating Support
- [v2/pkg/protocols/common/interactsh](./v2/pkg/protocols/common/interactsh) - Interactsh integration
- [v2/pkg/protocols/common/generators](./v2/pkg/protocols/common/generators) - Payload support for Requests (Sniper, etc.)
- [v2/pkg/protocols/common/executer](./v2/pkg/protocols/common/executer) - Default Template Executer
- [v2/pkg/protocols/common/replacer](./v2/pkg/protocols/common/replacer) - Template replacement helpers
- [v2/pkg/protocols/common/helpers/eventcreator](./v2/pkg/protocols/common/helpers/eventcreator) - Result event creator
- [v2/pkg/protocols/common/helpers/responsehighlighter](./v2/pkg/protocols/common/helpers/responsehighlighter) - Debug response highlighter
- [v2/pkg/protocols/common/helpers/deserialization](./v2/pkg/protocols/common/helpers/deserialization) - Deserialization helper functions
- [v2/pkg/protocols/common/hosterrorscache](./v2/pkg/protocols/common/hosterrorscache) - Host errors cache for tracking erroring hosts
- [v2/pkg/protocols/offlinehttp](./v2/pkg/protocols/offlinehttp) - Offline http protocol
- [v2/pkg/protocols/http](./v2/pkg/protocols/http) - HTTP protocol
- [v2/pkg/protocols/http/race](./v2/pkg/protocols/http/race) - HTTP Race Module
- [v2/pkg/protocols/http/raw](./v2/pkg/protocols/http/raw) - HTTP Raw Request Support
- [v2/pkg/protocols/headless](./v2/pkg/protocols/headless) - Headless Module
- [v2/pkg/protocols/headless/engine](./v2/pkg/protocols/headless/engine) - Internal Headless implementation
- [v2/pkg/protocols/dns](./v2/pkg/protocols/dns) - DNS protocol
- [v2/pkg/projectfile](./v2/pkg/projectfile) - Project File Implementation