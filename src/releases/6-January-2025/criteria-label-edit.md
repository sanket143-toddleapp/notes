# Criteria label edit

### CATEGORY_PLACEHOLDER criteria sets in source organizations
- Already exists in source organizations


### CATEGORY_PLACEHOLDER criteria sets in requested orgs
```sql
select 
  * 
from 
  auth.organization as ao 
where 
  ao.id in (
    '1123', '3562', '3639', '4020', '5455', 
    '5456', '4589', '933', '1300', '68', 
    '3826', '1267', '3749', '536870997', 
    '4002'
  )
```

```toml
eu-west-1: ["1300", "933", "3826"]
us-east-1: ["536870997"]
me-central-1: ["3749", "4020"]
ap-southeast-1: ["3639", "68", "1267", "4002", "1123", "4589", "5455", "5456", "3562"]
```


```sql
do $$ declare
temp_id bigint;

curriculum_program_id bigint;
curriculum_program_ids bigint[];

begin

raise notice 'porting started!';

select
 coalesce(array_agg(distinct cp.id),
 '{}')
into
 curriculum_program_ids
from
 planner.curriculum_program cp
 where cp.organization_id in (
    '1123', '3562', '3639', '4020', '5455', 
    '5456', '4589', '933', '1300', '68', 
    '3826', '1267', '3749', '536870997', 
    '4002'
 )
and cp.is_deleted = false;

foreach curriculum_program_id in array curriculum_program_ids loop
  with source_acs as (
 select
  *
 from
  json_populate_recordset(
         null :: record,
  '
[
   {"preset_set_id":193614527495939249,"label":"Score","d_order":1,"sub_type":"SCORE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"IB_PYP"}
  ,{"preset_set_id":193614528636789939,"label":"Score","d_order":1,"sub_type":"SCORE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"IB_MYP"}
  ,{"preset_set_id":193614528628401330,"label":"Grade","d_order":1,"sub_type":"GRADE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"IB_MYP"}
  ,{"preset_set_id":193614532004816053,"label":"Score","d_order":1,"sub_type":"SCORE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"UBD"}
  ,{"preset_set_id":193614531996427444,"label":"Grade","d_order":1,"sub_type":"GRADE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"UBD"}
  ,{"preset_set_id":193614532466189495,"label":"Score","d_order":1,"sub_type":"SCORE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"IB_DP"}
  ,{"preset_set_id":193614532457800886,"label":"Grade","d_order":1,"sub_type":"GRADE","criteria_type":"CATEGORY_PLACEHOLDER","fk_curriculum_uid":"IB_DP"}]' :: json
       ) as x(
     preset_set_id bigint,
  label text,
  d_order int,
  sub_type text,
  criteria_type criteria_type_enum,
  fk_curriculum_uid text
       )
  ),
  old_acs as (
 select
  acs.*
 from
  planner.curriculum_program cp
 join planner.curriculum_program_entity_map as cpem_acs on
  cpem_acs.fk_curriculum_program_id = cp.id
  and cpem_acs.entity_type = 'CRITERIA_SET'
  and cpem_acs.is_deleted = false
 inner join planner.academic_criteria_set acs on
  acs.criteria_type = 'CATEGORY_PLACEHOLDER'
  and acs.id::text = cpem_acs.fk_entity_id
  and acs.is_deleted = false
 where
  cp.id = curriculum_program_id
  ),
  new_acs as (
 select
  sa.label,
  sa.criteria_type,
  sa.d_order,
  cp.organization_id as organization_id,
  array_agg(distinct cpem_g.fk_entity_id) as grades,
  sa.sub_type,
  sa.preset_set_id as preset_set_id
 from
  source_acs as sa
 join planner.curriculum_program cp on
  cp.fk_curriculum_uid = sa.fk_curriculum_uid
  and cp.id = curriculum_program_id
 join planner.curriculum_program_entity_map as cpem_g on
  cpem_g.fk_curriculum_program_id = cp.id
  and cpem_g.entity_type = 'GRADE'
  and cpem_g.is_deleted = false
 join auth.org_grade og on
  og.uid = cpem_g.fk_entity_id
  and og.is_deleted = false
  and og.organization_id = cp.organization_id
 left join old_acs oa on
  oa.criteria_type = sa.criteria_type
  and oa.sub_type = sa.sub_type
 where
  oa.id is null
 group by
  sa.label,
  sa.criteria_type,
  sa.d_order,
  sa.sub_type,
  sa.preset_set_id,
  cp.organization_id
  ), acs as (
 insert into planner.academic_criteria_set (
     label,
  criteria_type,
  d_order,
  organization_id,
  grades,
  sub_type,
  preset_set_id,
  is_show_in_pr_criteria_sheet
 )
 select
  new_acs.label,
  new_acs.criteria_type,
  new_acs.d_order,
  new_acs.organization_id,
  new_acs.grades,
  new_acs.sub_type,
  new_acs.preset_set_id,
  false
 from
  new_acs returning *
  )
  insert into
 planner.curriculum_program_entity_map (
    fk_curriculum_program_id,
 fk_entity_id,
 entity_type,
 organization_id
  )
  select
 curriculum_program_id as fk_curriculum_program_id,
 acs.id as fk_entity_id,
 'CRITERIA_SET' as entity_type,
 acs.organization_id
  from
 acs;

raise notice 'ported (%)', curriculum_program_id;
end loop;

raise notice 'porting done!';
end;

$$;
```


### Create new base templates [eu-west-1]
```graphql
mutation enableFeatureInProgressReport($templateMapping: JSON) {
  documentation {
    enableFeatureInProgressReport(
      input: {
        featureTypes: [CREATE_BASE_TEMPLATE_COPIES]
        templateMapping: $templateMapping
      }
    )
  }
}  
```
```json
{
  "templateMapping": [
    {
      "sourceBaseTemplateId": "prBase1.0.dp.limited_release",
      "targetBaseTemplateId": "prBase1.0.dp.limited_release.acs_edit_x_projects"
    },
    {
      "sourceBaseTemplateId": "prBase1.0.myp.limited_release",
      "targetBaseTemplateId": "prBase1.0.myp.limited_release.acs_edit_x_projects"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_dp.limited_release",
      "targetBaseTemplateId": "prBase3.0.ib_dp.limited_release.acs_edit_x_projects"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_myp.limited_release",
      "targetBaseTemplateId": "prBase3.0.ib_myp.limited_release.acs_edit_x_projects"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_pyp",
      "targetBaseTemplateId": "prBase3.0.ib_pyp.limited_release.acs_edit"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_ubd",
      "targetBaseTemplateId": "prBase3.0.ib_ubd.limited_release.acs_edit"
    },
    {
      "sourceBaseTemplateId": "prBase1.0.myp.score.limited_release",
      "targetBaseTemplateId": "prBase1.0.myp.score.limited_release.acs_edit_x_projects"
    },
    {
      "sourceBaseTemplateId": "prBase2.0.ib_pyp_score",
      "targetBaseTemplateId": "prBase2.0.ib_pyp_score.limited_release.acs_edit"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_myp",
      "targetBaseTemplateId": "prBase3.0.ib_myp.limited_release.acs_edit"
    },
    {
      "sourceBaseTemplateId": "prBase3.0.ib_dp",
      "targetBaseTemplateId": "prBase3.0.ib_dp.limited_release.acs_edit"
    }
  ]
}
```
```
990
```

### Map base template with correct base template
```sql
select 
  ao.id, 
  prbt.id, 
  prbt.body ->> 'projectsVersion' as project_version 
from 
  auth.organization as ao 
  join documentation.map_progress_report_base_template_organization as mprbto on mprbto.organization_id = ao.id 
  and mprbto.is_deleted = false 
  join documentation.progress_report_base_template as prbt on prbt.id = mprbto.fk_progress_report_base_template_id 
  and prbt.is_deleted = false 
where 
  ao.id in (
    '1123', '3562', '3639', '4020', '5455', 
    '5456', '4589', '933', '1300', '68', 
    '3826', '1267', '3749', '536870997', 
    '4002'
  );
```

```sql
with new_base_templates as (
select
 case when prbt.body->>'projectsVersion' = '2.0'
 then concat(replace(prbt.id, '.limited_release', ''), '.limited_release.acs_edit_x_projects')
 else concat(prbt.id, '.limited_release.acs_edit') end
 as new_id,
 prbt.id as old_id,
 prbt.name, prbt.body,
 prbt.is_deleted,
 prbt.type,
 prbt.grading_period_type,
 mprbto.id as mprbto_id,
 mprbto.organization_id
from documentation.progress_report_base_template prbt
join documentation.map_progress_report_base_template_organization mprbto on mprbto.fk_progress_report_base_template_id = prbt.id
where prbt.grading_period_type = 'REPORTING'
and mprbto.organization_id in (
    '1123', '3562', '3639', '4020', '5455',
    '5456', '4589', '933', '1300', '68',
    '3826', '1267', '3749', '536870997',
    '4002'
  )
)
update documentation.map_progress_report_base_template_organization mprbto
set fk_progress_report_base_template_id = nbt.new_id
from new_base_templates as nbt
where mprbto.id = nbt.mprbto_id;
```

### Port base templates
```graphql
mutation portTemplates {
  documentation {
    portProgressReportTemplates(
      input: {
        portingEntityType: BASE_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
        featureTypes: [EDIT_CRITERIA_SET_LABEL]
        gradingPeriodTypes: [REPORTING]
        progressReportBaseTemplateIds: [
          "prBase1.0.dp.limited_release.acs_edit_x_projects",
          "prBase1.0.myp.limited_release.acs_edit_x_projects",
          "prBase3.0.ib_dp.limited_release.acs_edit_x_projects",
          "prBase3.0.ib_myp.limited_release.acs_edit_x_projects",
          "prBase3.0.ib_pyp.limited_release.acs_edit",
          "prBase3.0.ib_ubd.limited_release.acs_edit",
          "prBase1.0.myp.score.limited_release.acs_edit_x_projects",
          "prBase2.0.ib_pyp_score.limited_release.acs_edit",
          "prBase3.0.ib_myp.limited_release.acs_edit",
          "prBase3.0.ib_dp.limited_release.acs_edit"
        ]
        shouldUpdateBaseTemplateIdMapping: false
      }
    )
  }
}
```

 
"1123", "3562", "3639", "4020", "5455", "5456", "4589", "933", "1300", "68", "3826", "1267", "3749", "536870997", "4002"

### Get target base template id mapping from following query
```sql
with t as (select 
  ao.id, json_agg(json_build_object('curriculumType', prbt.fk_curriculum_uid, 'newBaseTemplateId', prbt.id)) as base_template_ids
from 
  auth.organization as ao 
  join documentation.map_progress_report_base_template_organization as mprbto
  on mprbto.organization_id = ao.id
  join documentation.progress_report_base_template as prbt
  on prbt.id = mprbto.fk_progress_report_base_template_id
  and prbt.grading_period_type = 'REPORTING'
  and prbt.is_deleted = false
where 
  ao.id in (
    '1123', '3562', '3639', '4020', '5455', 
    '5456', '4589', '933', '1300', '68', 
    '3826', '1267', '3749', '536870997', 
    '4002'
  )
group by ao.id)
select json_build_object('organizationId', t.id::text, 'targetBaseTemplateIdMapping', t.base_template_ids) from t;
```

**Port generated progress report template**

```graphql
mutation portTemplates {
  documentation {
    portProgressReportTemplates(
      input: {
        portingEntityType: GENERATED_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
        featureTypes: [EDIT_CRITERIA_SET_LABEL]
        gradingPeriodTypes: [REPORTING]
        organizationIds: ["4005"]
        curriculumTypes: [IB_MYP, IB_DP, IB_PYP, UBD]
        shouldUpdateBaseTemplateIdMapping: true
        targetBaseTemplateIdMapping: [
          {
            curriculumType: IB_PYP
            newBaseTemplateId: "prBase1.0.limited_release.acs_edit"
          }
          {
            curriculumType: IB_MYP
            newBaseTemplateId: "prBase1.0.myp.limited_release.acs_edit"
          }
          {
            curriculumType: IB_DP
            newBaseTemplateId: "prBase1.0.dp.limited_release.acs_edit"
          }
          {
            curriculumType: UBD
            newBaseTemplateId: "prBase1.0.ubd.limited_release.acs_edit"
          }
        ]
      }
    )
  }
}
```

#### Enable feature flag
```graphql
mutation {
  documentation {
    enableFeatureInProgressReport(
      input: {
        organizationIds: ["4005"]
        featureTypes: [EDIT_CRITERIA_SET_LABEL_PERMISSION_UPDATE]
      }
    )
  }
}
```

```toml
eu-west-1: ["1300", "933", "3826"]
us-east-1: ["536870997"]
me-central-1: ["3749", "4020"]
ap-southeast-1: ["3639", "68", "1267", "4002", "1123", "4589", "5455", "5456", "3562"]
```
