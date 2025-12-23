# APIM Migration Project

I am a senior application architect, and I am currently working on a migration project. Details are below.

## Current Application Overview

I have a Spring Boot application located at `c:\projects\prodigy_code2\blackbookvalue_apim`.

This application provides REST endpoints that retrieve BlackBookValue from an external website.

The example is the following endpoint inside `BbvRestController.java`:

```java
@PostMapping("/uvc")
public BbvDTO retrieveBlackBookValueByUvc(
        @RequestParam String uvc,
        @RequestParam(required = false) Integer mileage) {
    return bbvRestService.retrieveBlackBookValueByUvc(policyLineId, uvc, mileage, userId);
}
```

For simplicity, we will call this Spring Boot application as **APP**.

## Current Process Flow

This endpoint performs the following tasks:

### When a client request calls our endpoint: @PostMapping("/uvc")

1. **APP** will try to access the external REST endpoint **ENDPOINT_A** to get a valid token with username and password. (username and password are configured inside application.properties, example like below)
2. **APP** will use this valid token (from step 1) to query an external GraphQL endpoint **ENDPOINT_B** to get a response string
   - If `mileage == 0`, then use `blackbookvalue.api.graphql.query` (configured in application.properties) as the query String to call the GraphQL endpoint.
   - If `mileage != 0`, then use `blackbookvalue.api.graphql.querywithmileage` (configured in application.properties) as the query String to call the GraphQL endpoint.
3. **APP** converts the string to object BbvDTO.
4. **APP** will return this object BbvDTO to the client.

### Implementation Details

- For how **APP** accesses external endpoint **ENDPOINT_A**, the actual code is located in `TokenManager.java`
- For how **APP** accesses external endpoint **ENDPOINT_B**, the actual code is located in `GraphQLExecutor.java`

### Configuration

The actual configurations for accessing both endpoints are inside `application.properties`. Also code for access GraphQL site is as below. Code to access REST **ENDPOINT_A** is simple. Below is the extract:

```java
public List<BbvDTO> fetchByVinUvc(String uvc, int mileage) {
    String document = (mileage == 0) ? bbvProperties.getQuery() : bbvProperties.getQuerywithmileage();
    Map<String, Object> variables = getVariables(identifier, mileage);
    List<BbvDTO> result = graphQLExecutor.executeAndMap(document, variables,
                                                        bbvProperties.getPath(), BbvDTO.class,
                                                        tokenService.getValidToken());
    validateResult(result, vin, uvc, mileage);
    return result;
}

private Map<String, Object> getVariables(String identifier, int mileage) {
    Map<String, Object> variables = new HashMap<>();
    variables.put("uvc", identifier);
    variables.put("mileage", mileage);
    return variables;
}

public class GraphQLExecutor {

    private final HttpGraphQlClient graphQLClient;
    BbvApiServiceProperties bbvProperties;

    public GraphQLExecutor(WebClient.Builder webClientBuilder, BbvApiServiceProperties bbvProperties) {
        WebClient webClient = webClientBuilder
                .baseUrl(bbvProperties.getUrl())
                .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                .build();

        this.graphQLClient = HttpGraphQlClient.builder(webClient).build();
        this.bbvProperties = bbvProperties;
    }

    public <T> List<T> executeAndMap(String document, Map<String, Object> variables, String path, Class<T> targetType, String token) {
        log.debug("Calling API={}", document);
        try {
            return graphQLClient
                    .mutate()
                    .headers(headers -> headers.set(HttpHeaders.AUTHORIZATION, "Token " + token))
                    .build()
                    .document(document)
                    .variables(variables)
                    .retrieve(path)
                    .toEntityList(targetType)
                    .block();
        } catch (Exception e) {
            log.error("GraphQL query execution failed: {}", e.getMessage(), e);
            throw new AppErrorException("GraphQL query execution failed", e);
        }
    }
}
```

```properties
blackbookvalue.api.login.url=https://service.externalsite.com/UsedCarWS/UsedCarWS/Token/Get?
blackbookvalue.api.login.username=${BLACKBOOKVALUE_API_LOGIN_USERNAME}
blackbookvalue.api.login.password=${BLACKBOOKVALUE_API_LOGIN_PASSWORD}

blackbookvalue.api.login.logouturl=https://service.externalsite.com/UsedCarWS/UsedCarWS/Token/Expire?

blackbookvalue.api.graphql.url=https://service.externalsite.com/UsedCarWS/UsedCarWS/GraphQL

blackbookvalue.api.graphql.path=usedvehicles.usedvehicles
blackbookvalue.api.graphql.querywithmileage=query($vin: String!, $mileage: Int) { usedvehicles(vin: $vin, mileage: $mileage) { usedvehicles { uvc vin model_year make model series style base_whole_avg fuel_type hwy_mpg city_mpg mileage_list { range_begin range_end xclean clean avg rough finadv } } } }
blackbookvalue.api.graphql.query=query($vin: String!) { usedvehicles(vin: $vin) { usedvehicles { uvc vin model_year make model series style base_whole_avg fuel_type hwy_mpg city_mpg mileage_list { range_begin range_end xclean clean avg rough finadv } } } }
```

Please check above existing process.

## New Requirement: Migration to Azure APIM

Now we have a new requirement: we want to migrate the access to both external endpoints from this Spring Boot application **APP** to Azure APIM.

### Proposed Process Flow

The process will be as follows:

### When a client request calls our endpoint: @PostMapping("/uvc")

1. **APP** will call an APIM REST endpoint (needs to be implemented) to retrieve the black book value object.
   - 1.1 APIM will either take the Token from caching or query the external REST endpoint **ENDPOINT_A** to get a valid token.
   - 1.2 APIM will use this valid token to query an external GraphQL **ENDPOINT_B** to get a response string, return to the **APP**.
   - 1.3 **APP** will convert it to object.
2. **APP** will return the object to the client.

## Next Steps

Could you please read the requirement carefully and repeat the requirement first?

After I confirm the requirement, then please provide me a step-by-step plan on how to achieve this goal:

1. **What needs to be done in APIM**: Detailed step-by-step implementation plan I can use to implement this inside APIM.

First, please let me know if you have any questions.

---

## Reply from Opus 4.5

### Understanding Your Requirements

Thank you for the detailed explanation. Let me summarize my understanding of your migration project:

### Current State Summary

#### Application Overview

- **APP**: A Spring Boot application that provides REST endpoints for retrieving BlackBookValue data
- **Location**: `c:\projects\prodigy_code2\blackbookvalue_apim`

#### Current Process Flow

```
┌────────┐      ┌─────────┐      ┌─────────────┐      ┌─────────────┐
│ Client │ ───► │   APP   │ ───► │ ENDPOINT_A  │      │ ENDPOINT_B  │
└────────┘      │ (Spring │      │ (REST/Token)│      │ (GraphQL)   │
                │  Boot)  │ ───► │             │      │             │
                └─────────┘      └─────────────┘      └─────────────┘
```

**Step-by-step:**

1. Client calls `POST /uvc` with parameters: `uvc`, `mileage` (optional)
2. APP calls **ENDPOINT_A** (`https://service.externalsite.com/.../Token/Get?`) with username/password to obtain a valid token
3. APP calls **ENDPOINT_B** (`https://service.externalsite.com/.../GraphQL`) with the token:
   - If `mileage == 0`: Uses `blackbookvalue.api.graphql.query`
   - If `mileage != 0`: Uses `blackbookvalue.api.graphql.querywithmileage`
4. APP converts the GraphQL response to BbvDTO object
5. APP returns BbvDTO to the client

#### Key Implementation Files

| File | Purpose |
|------|---------|
| `BbvRestController.java` | REST endpoint definition |
| `TokenManager.java` | Token retrieval logic for ENDPOINT_A |
| `GraphQLExecutor.java` | GraphQL execution logic for ENDPOINT_B |
| `application.properties` | Configuration (URLs, credentials, GraphQL queries) |

### Target State (After Migration)

#### Proposed Process Flow

```
┌────────┐      ┌─────────┐      ┌────────────┐      ┌─────────────┐      ┌─────────────┐
│ Client │ ───► │   APP   │ ───► │ Azure APIM │ ───► │ ENDPOINT_A  │      │ ENDPOINT_B  │
└────────┘      │ (Spring │      │            │      │ (REST/Token)│      │ (GraphQL)   │
                │  Boot)  │      │            │ ───► │             │      │             │
                └─────────┘      └────────────┘      └─────────────┘      └─────────────┘
```

**Step-by-step:**

1. Client calls `POST /uvc` with parameters: `uvc`, `mileage` (optional)
2. APP calls a new APIM REST endpoint (to be created)
3. APIM handles:
   - 3.1 Token management: Retrieve from cache OR call **ENDPOINT_A** for a fresh token
   - 3.2 Call **ENDPOINT_B** (GraphQL) with the valid token
   - 3.3 Return the response string to APP
4. APP converts the response to BbvDTO object
5. APP returns BbvDTO to the client

### Clarifying Questions

Before I provide the detailed implementation plan, I have a few questions:

#### Token Management

1. What is the token format returned from **ENDPOINT_A**? (e.g., JWT, opaque string, etc.)
2. What is the token expiration time/TTL? (This is critical for caching strategy in APIM)
3. How does the current `TokenManager.java` determine if a token is expired? (Is there a specific field in the response, or is it time-based?)

#### APIM Configuration

1. Is Azure APIM already provisioned in your Azure environment, or does it need to be created?
2. What APIM tier are you using or planning to use? (Consumption, Developer, Basic, Standard, Premium) - This affects available features like caching.

#### Security & Authentication

1. How should APP authenticate to APIM? (Subscription key, OAuth 2.0, Managed Identity, etc.)
2. Where should the BlackBook credentials (username/password) be stored in Azure? (APIM Named Values, Azure Key Vault, etc.)

#### Response Handling

1. Should APIM return the raw GraphQL response, or should it transform/extract specific data before returning to APP?
2. What error handling behavior is expected? (e.g., if **ENDPOINT_A** fails, if token is invalid, if GraphQL returns errors)

#### Additional Requirements

1. Are there any rate limiting, throttling, or logging requirements for this APIM endpoint?
2. Do you need to support both GraphQL queries (with and without mileage) through a single APIM endpoint, or separate endpoints?


### request 2
#### Token Management

1. the token format returned from **ENDPOINT_A** is a standard string.
2. The token expiration time/TTL? let us put it as 6 hours for now.
3. it is simplw time based check to determine if a token is expired. 

#### APIM Configuration

1. Azure APIM already provisioned is already created.
2. What APIM tier are you using or planning to use? it is Standard 

#### Security & Authentication

1. How should APP authenticate to APIM? (Subscription key, OAuth 2.0, Managed Identity, etc.)
2. Where should the BlackBook credentials (username/password) be stored in Azure? Azure Key Vault

#### Response Handling

1. Should APIM return the raw GraphQL response, transform it to string before returning to APP
2. What error handling behavior is expected? (e.g., if **ENDPOINT_A** fails, if token is invalid, if GraphQL returns errors): raise exception with error message.

#### Additional Requirements

1. Are there any rate limiting, throttling, or logging requirements for this APIM endpoint?
    rate limiting and normal logging for request and response only. 
2. Do you need to support both GraphQL queries (with and without mileage) through a single APIM endpoint, or separate endpoints? With Separated endpoint. 
