# Projects v2

NOTE: Downloads will be released on 31st Jan

### Cherry picks
- Porting update to add curriculum type check (@parakh is doing) [14444@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14444)
- Make the Projects v2 feature flag default true [14500@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14500)
- Put Downloads on different feature flag

### To check
- Projects criteria sets are already created

### Port base templates
- All the base templates for DP_PROJECTS_REVAMP

**Includes criteria label edit porting also**

```graphql
mutation portTemplates {
  documentation {
    portProgressReportTemplates(
      input: {
        portingEntityType: BASE_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
        curriculumTypes: [IB_DP, IB_MYP, IB_PYP, UBD]
        progressReportBaseTemplateIds: []
        featureTypes: [DP_PROJECTS_REVAMP, MYP_PROJECTS_REVAMP, EDIT_CRITERIA_SET_LABEL]
      }
    )
  }
}
```

### Generated progress report templates
- All the generated templates for MYP_PROJECTS_REVAMP
- Will there be any issues in re-porting the old progress report template
- Preserve tokComment

```graphql
mutation portTemplates {
  documentation {
    portProgressReportTemplates(
      input: {
        portingEntityType: GENERATED_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
        featureTypes: [DP_PROJECTS_REVAMP, MYP_PROJECTS_REVAMP, EDIT_CRITERIA_SET_LABEL]
        gradingPeriodTypes: [REPORTING]
        curriculumTypes: [IB_MYP, IB_DP, IB_PYP, UBD]
        shouldUpdateBaseTemplateIdMapping: false
      }
    )
  }
}
```
### Update ChatGPT prompt set update
- Migration to update org group prompt set mapping (@vimal)

### Feature flag update
- Migration cherry-pick to make default true for (@vimal)





