# GraphQL Backend

### Currently
- No bulk upload
- No bulk download
- No subjects mapping, just grades and created by
- Currently, admin doesn't have his personal comments, all the comments created by
admin are global comments (school level)
- 3 filters for teacher (All, Me, Admin)
  - All -> `is_global = null`
  - Me -> `is_global = false`
  - Admin -> `is_global = true`

### Comment creation
```graphql
mutation createNewCommentTemplate(
  $input: ProgressReportCommentTemplateCreateInput!
) {
  documentation {
    createProgressReportCommentTemplate(input: $input) {
      id
      title
      message
      isGlobal
      createdBy {
        id
      }
      grades {
        id
        name
      }
    }
    __typename
  }
}
```
```json
{
	"input": {
		"curriculumProgramId": "1887746213821170",
		"grades": [
			"1887758620564787",
			"1887758620564788"
		],
		"isGlobal": true,
		"message": "Test creation",
		"title": "Test"
	}
}
```

### Comments fetching
- Paginated

#### On admin end
```graphql
query getAllCommentTemplates(
  $filters: ProgressReportCommentTemplateFilterOptions!
) {
  documentation {
    progressReportCommentTemplates(filters: $filters) {
      pageInfo {
        hasNextPage
        hasPreviousPage
        startCursor
        endCursor
        __typename
      }
      totalCount
      edges {
        cursor
        node {
          id
          title
          message
          isGlobal
          createdBy {
            id
            __typename
          }
          grades {
            id
            name
            __typename
          }
          __typename
        }
        __typename
      }
      __typename
    }
    __typename
  }
}
```
```json
{
	"filters": {
		"curriculumProgramIds": [
			"1887746213821170"
		],
		"first": 20,
		"isGlobal": true,
		"searchText": ""
	}
}
```


### Publish comment bank [Maple bear]

`FF: FeatureFlag:PublishCommentBank, FeatureFlag:PublishCommentBankUpdate`

We will have to handle subjects mapping and any new feature added in the comment bank

```graphql
mutation publishCommentBank($input: PublishCommentBankInput!) {
  documentation {
    publishCommentBank(input: $input)
    __typename
  }
}
```
```json
{
	"input": {
		"organizationId": "2615",
		"shouldUpdateComments": true
	}
}
```
