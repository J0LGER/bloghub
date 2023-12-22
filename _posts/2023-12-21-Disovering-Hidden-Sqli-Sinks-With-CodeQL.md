---
layout: default
title:  "Discovering Hidden Sqli Sinks With CodeQL"
date:   2023-09-20 11:12:00 -0500
categories: jekyll update
templateEngineOverride: md
permalink: /bloghub/Disovering-Hidden-Sqli-Sinks-With-CodeQL/
---

{% raw %} 

## Introduction

I came across an interesting framework in one of my engagements where the codebase was pretty old, a microfinance framework by the Apache community called *Fineract,* described as “a framework to maintain and enhance a cloud-ready core banking system for robust, scalable, and secure operations of financial institutions.”, the versions before `≤ 1.8.4` are vulnerable to authenticated SQL injection. We will utilize CodeQL capabilities in identifying flow paths that lack validation calls, this article assumes you have basic knowledge of using CodeQL and its query development. 

## The First Sight: The Attack Surface

When you start researching against any code base, you start by surface-scratching the frameworks, libraries, design patterns, and the architecture you’re trying to test, fortunately, I had a running deployment so we can skip the “good luck deploying the app” part and dive directly to our code review. 

A quick *find* command reveals `.java` files we are interested in so we can narrow down our attack surface.

```bash
$> find . -name "*.java" | grep -v "/src/test"

./fineract-provider/src/main/java/org/apache/fineract/**organisation**/office/exception/OfficeTransactionNotFoundException.java
./fineract-provider/src/main/java/org/apache/fineract/**organisation**/office/exception/InvalidOfficeException.java
./fineract-provider/src/main/java/org/apache/fineract/**organisation**/office/exception/RootOfficeParentCannotBeUpdated.java
./fineract-provider/src/main/java/org/apache/fineract/**organisation**/office/exception/OfficeNotFoundException.java
./fineract-provider/src/main/java/org/apache/fineract/ServerApplication.java
./fineract-provider/src/main/java/org/apache/fineract/**commands**/data/AuditSearchData.java
./fineract-provider/src/main/java/org/apache/fineract/**commands**/data/ProcessingResultLookup.java
```

Inspecting the tree view of each folder we have an insight into how the APIs and their corresponding handlers are designed  

```bash
$> tree -L 2 ./fineract-provider/src/main/java/org/apache/fineract/commands
./fineract-provider/src/main/java/org/apache/fineract/commands
├── annotation
│   └── CommandType.java
├── api
│   ├── AuditsApiResource.java
│   ├── MakercheckersApiResource.java
│   └── MakercheckersApiResourceSwagger.java
├── data
│   ├── AuditData.java
│   ├── AuditSearchData.java
│   └── ProcessingResultLookup.java
├── domain
│   ├── CommandProcessingResultType.java
│   ├── CommandSource.java
│   ├── CommandSourceRepository.java
│   └── CommandWrapper.java
├── exception
│   ├── CommandNotAwaitingApprovalException.java
│   ├── CommandNotFoundException.java
│   ├── RollbackTransactionAsCommandIsNotApprovedByCheckerException.java
│   └── UnsupportedCommandException.java
├── handler
│   └── NewCommandSourceHandler.java
├── provider
│   └── CommandHandlerProvider.java
└── service
    ├── AuditReadPlatformServiceImpl.java
    ├── AuditReadPlatformService.java
    ├── CommandProcessingService.java
    ├── CommandWrapperBuilder.java
    ├── PortfolioCommandSourceWritePlatformServiceImpl.java
    ├── PortfolioCommandSourceWritePlatformService.java
    └── SynchronousCommandProcessingService.java
```

API mappings are defined inside the `api` directory, meanwhile, their controller interface and implementation functions can be found inside the **service** directory.

## **Delving into the Details**

In modern applications, a researcher may start searching for raw query executions if present since they generally utilize Object Relation Mapping (ORMs) libraries that prevent any SQLi occurrences if used properly, unless in other cases, for example when raw queries are used. Raw queries introduce interesting attack surfaces, however, since our facing framework is considered an old project, no ORMs were in use. instead, we can detect several direct query executions with string concatenation in several classes. 

The `/groups` mapping, for instance, with the usage of `javax` library, provides a set of annotations to define a new mapping, the `GroupsApiResource` class and its methods will define all route paths and their handlers for routes with `/gourps/*` as its prefix.

```bash
@Path("/groups")
@Component
@Scope("singleton")
@Tag(name = "Groups", description = "Groups are used to provide a distinctive banking distribution channel used in microfinances throughout the world. The Group is an administrative unit. It can contain as few as 5 people or as many as 40 depending on how its used.\n"
        + "\n"
        + "Different styles of group lending - Joint-Liability Group, Grameen Model (Center-Group), Self-Help Groups, Village/Communal Banks)")
public class GroupsApiResource {

    private final PlatformSecurityContext context;
    private final GroupReadPlatformService groupReadPlatformService;
    private final CenterReadPlatformService centerReadPlatformService;
    private final ClientReadPlatformService clientReadPlatformService;
    private final ToApiJsonSerializer<Object> toApiJsonSerializer;
.........
```

The index `/` path of GET mapping reveals interesting query parameters:

- offset
- limit
- orderBy
- sortOrder

These parameters are often called Pagination Parameters in the Development world, they also indicate a possible query execution.

```java
@GET
    @Consumes({ MediaType.APPLICATION_JSON })
    @Produces({ MediaType.APPLICATION_JSON })
    @Operation(summary = "List Groups", description = "The default implementation of listing Groups returns 200 entries with support for pagination and sorting. Using the parameter limit with description -1 returns all entries.\n\n"
            + "Example Requests:\n\n" + "\n\n" + "groups\n\n" + "\n\n" + "groups?fields=name,officeName,joinedDate\n\n" + "\n\n"
            + "groups?offset=10&limit=50\n\n" + "\n\n" + "groups?orderBy=name&sortOrder=DESC")
    @ApiResponses({
            @ApiResponse(responseCode = "200", description = "OK", content = @Content(schema = @Schema(implementation = GroupsApiResourceSwagger.GetGroupsResponse.class))) })
    public String retrieveAll(@Context final UriInfo uriInfo,
            @QueryParam("officeId") @Parameter(description = "officeId") final Long officeId,
            @QueryParam("staffId") @Parameter(description = "staffId") final Long staffId,
            @QueryParam("externalId") @Parameter(description = "externalId") final String externalId,
            @QueryParam("name") @Parameter(description = "name") final String name,
            @QueryParam("underHierarchy") @Parameter(description = "underHierarchy") final String hierarchy,
            @QueryParam("paged") @Parameter(description = "paged") final Boolean paged,
            @QueryParam("offset") @Parameter(description = "offset") final Integer offset,
            @QueryParam("limit") @Parameter(description = "limit") final Integer limit,
            @QueryParam("orderBy") @Parameter(description = "orderBy") final String orderBy,
            @QueryParam("sortOrder") @Parameter(description = "sortOrder") final String sortOrder,
            @QueryParam("orphansOnly") @Parameter(description = "orphansOnly") final Boolean orphansOnly) {

        this.context.authenticatedUser().validateHasReadPermission(GroupingTypesApiConstants.GROUP_RESOURCE_NAME);
        final **PaginationParameters** parameters = PaginationParameters.instance(paged, offset, limit, orderBy, sortOrder);
        final ApiRequestJsonSerializationSettings settings = this.apiRequestParameterHelper.process(uriInfo.getQueryParameters());

        final SearchParameters searchParameters = SearchParameters.forGroups(officeId, staffId, externalId, name, hierarchy, offset, limit,
                orderBy, sortOrder, orphansOnly);
        if (parameters.isPaged()) {
            final Page<GroupGeneralData> groups = this.groupReadPlatformService.retrievePagedAll(searchParameters, parameters);
            return this.toApiJsonSerializer.serialize(settings, groups, GroupingTypesApiConstants.GROUP_RESPONSE_DATA_PARAMETERS);
        }

        final Collection<GroupGeneralData> groups = this.groupReadPlatformService.retrieveAll(searchParameters, parameters);
        return this.toApiJsonSerializer.serialize(settings, groups, GroupingTypesApiConstants.GROUP_RESPONSE_DATA_PARAMETERS);
    }
```

After our targeted parameters are serialized into **PaginationParameters** object instance, they are passed to either **retrievePagedAll()** or **retrieveAll(),** depending on whether `paged` parameter was sent in the HTTP request, diving into the function body of `retrieveAll()` shows:

```java
@Override
    public Collection<GroupGeneralData> retrieveAll(SearchParameters searchParameters, final PaginationParameters parameters) {
        final AppUser currentUser = this.context.authenticatedUser();
        final String hierarchy = currentUser.getOffice().getHierarchy();
        final String hierarchySearchString = hierarchy + "%";

        final StringBuilder sqlBuilder = new StringBuilder(200);
        sqlBuilder.append("select ");
        sqlBuilder.append(this.allGroupTypesDataMapper.schema());
        final SQLBuilder extraCriteria = getGroupExtraCriteria(this.allGroupTypesDataMapper.schema(), searchParameters);
        extraCriteria.addCriteria("o.hierarchy like ", hierarchySearchString);

        sqlBuilder.append(" ").append(extraCriteria.getSQLTemplate());

        if (searchParameters != null) {
            if (searchParameters.isOrphansOnly()) {
                sqlBuilder.append(" and g.parent_id is NULL");
            }
        }

        if (parameters != null) {
            if (parameters.isOrderByRequested()) {
                sqlBuilder.append(parameters.orderBySql());
                this.columnValidator.validateSqlInjection(sqlBuilder.toString(), parameters.orderBySql());
            }
```

The function starts a StringBuilder variable that apparently builds up the select statement string, at the end of the snippet we see an *if statement* which checks the presence of the **isOrder** passed query parameter by calling its *getter* method on the previously mentioned serialized object, the parameter is eventually passed and appended to the StringBuilder object, however, we finally hit a validation function `validateSqlInjection()` that passes the final query string, this shows that the application may have an old record with SQL injection vulnerabilities, which may be resolved through this validation class.

Indeed, we can verify the SQL injection record by looking for past CVEs:

![cves](https://github.com/J0LGER/bloghub/assets/54769522/a796d82d-4a41-499e-9087-c63d84b9a461)


We’ll NOT get into the validation class for now, however, it detects any patterns outside a whitelist regex and raises exceptions if any were matched, however, bypassing this validation is out of the scope of this article. 

## **Harnessing CodeQL's Analytical Power for SQL Injection Detection**

CodeQL is a powerful tool that was developed by *Semmle* and acquired later by *Github*, it provides testers with a semantic code analysis engine that empowers them with the capability to execute queries against a codebase. 

In essence, it enables you to engage with the codebase much like you would with a relational database file, allowing you to execute select statements on your codebase. This approach aids in uncovering patterns that could potentially lead to the discovery of new vulnerabilities, or finding hidden vulnerability variations that may be hard to be detected using manual approaches. 

Our target is to detect any SQLi paths that miss the usage of the validation class by using *Taint Analysis*, developers may sometimes miss the need to remember to call validation functions before each case, resulting in new arising vulnerabilities. 

The CodeQL query shall achieve the following objectives: 

- Search for `RemoteFlowSources` and identify them as tainted sources.
- Filter out *String* type RemoteFlowSources and exclude other data types.
- Search for DB query execution functions provided by the `JdbcTemplate` class.
- Check if the node path is being passed as a first parameter to the previously mentioned query execution functions, if so, raise them as sinks.
- Define the `SQLInjectionValidator` class as a sanitizer so we limit false positive flow paths.

Note that I referred back to the `springframework` documentation to check for JdbcTemplate class methods that execute queries, such as *query()*, *update()*, *batchUpdate()*, etc ...

```java
import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.dataflow.TaintTracking
import DataFlow::PathGraph

class MyConfig extends TaintTracking::Configuration { 
  MyConfig() { this = "SQLi config for Apache Fineract v1.8.4" }

  override predicate isSource(DataFlow::Node source) {
    source.getType().toString().matches("String%") and
    exists(RemoteFlowSource s | source = s and
    not s.getLocation().getFile().getRelativePath().matches("%/src/test/%"))
  }

  override predicate isSink(DataFlow::Node sink) {
    exists(MethodAccess m | 
       m.getMethod() instanceof JdbcQueryFunction and 
       sink.asExpr() = m.getArgument(0))
  }
      
  override predicate isSanitizer(DataFlow::Node sanitizer) { 
    exists(MethodAccess m | 
      m.getMethod() instanceof SQLiSanitizer and 
      sanitizer.asExpr() = m.getArgument(0))
  }

}

class SQLiSanitizer extends Method { 
  SQLiSanitizer() { 
		this.getDeclaringType().hasQualifiedName("org.apache.fineract.infrastructure.security.utils","SQLInjectionValidator") and 
    (
		this.getName() = "validateSQLInput" or 
    this.getName() = "validateAdhocQuery"
	  )
	}
}

class JdbcQueryFunction extends Method { 
  JdbcQueryFunction() { 
    this.getDeclaringType().getQualifiedName().matches("%org.springframework.jdbc%") and
    (
    this.getName().matches("%query%") or 
    this.getName().matches("%update%") or
    this.getName() = "batchUpdate" or 
    this.getName() = "execute"
    )
  }
}

from MyConfig conf, DataFlow::PathNode source, DataFlow::PathNode sink
where
  conf.hasFlowPath(source, sink)
select 
  sink, source, "possible SQLi"
```

We obtained a considerable number of results. However, upon manual review, I discovered that they were duplicates. Therefore, we can focus on lines 20-24.

![codeql-1](https://github.com/J0LGER/bloghub/assets/54769522/39b995a0-f4ad-46e3-b160-5186bfc0723b)


The query parameters *orderBy* and *sortOrder* have been identified as potential SQL injection vulnerabilities. Upon checking the sink object, it was found that a query execution is performed against the *sqlBuilder* object. By tracing this object, it was determined that user input was passed without any validation and was contained within the `paginationParameters`.

![codeql-2](https://github.com/J0LGER/bloghub/assets/54769522/c2d5b942-e56e-4966-ad5a-7d8ef68f6855)


## Exploitation

The framework gives you the flexibility to connect to different *DBMSs* of your choice, and the one we are facing is using *PostgreSQL*, This will give us me more ways to strike in because of its stacked query capability.

To see if it's vulnerable, we can try a simple sleep injection payload:

![image](https://github.com/J0LGER/bloghub/assets/54769522/07b7a741-24f6-455c-a990-b87dd180c69b)


The response time reveals the execution of our injection verifying our `is_superuser` role, the vulnerability had two variants on the `/recurringdepositaccounts` and `/fixeddepositaccounts` APIs.

## Remediation

Such vulnerabilities arise when user input sinks to dangerous functions without proper parametrization, Despite the input validation and character escaping being a good secondary security layer, parametrization at first place helps with the defense-in-depth practice.

The vulnerability was fixed in earlier versions of the framework.

## Conclusion

When testing particularly large code bases, encountering a validation does not necessarily mean that the vulnerability is absent, to address this, we utilized CodeQL to identify different flow paths that do not pass through validation methods, in addition to its usage to detect all variants of the vulnerability. Although CodeQL can be used within your CI/CD for automated SAST detections, using it directly against a codebase with specific patterns and code scenarios shows the need for CodeQL and its capabilities.

{% endraw %}
