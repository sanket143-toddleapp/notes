# 10th January 2025 Release

| Feature                | Status  | PoC            | Remarks                                                           | Release status | DQC Doc |
|:-----------------------|:--------|:---------------|:------------------------------------------------------------------|:---------------|:--------|
| [Criteria label edit](../../releases/10-January-2025/criteria-lable-edit/global-release.md)    | Pending | Nipun          | Under testing                                                     | Pending        |         |
| Downloads enhancements | Pending | Kapil          | Under testing + Development                                       | Pending        |         |
| Projects v2            | Pending | Parakh + Vimal | Under testing + Downloads is in Development                       | Pending        |         |
| 1 Pager Transcript     | Pending | Shubham        | Testing?                                                          | Pending        |         |
| Grading period locking | Pending | Tanuj          | Gradebook team will release it but development pending on our end | Pending        |         |


No downloads for
- Projects v2
- Criteria label edit


### Create CATEGORY_PLACEHODLER criteria sets in all orgs
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
 where cp.is_deleted = false;

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

### Is ported false of all the base templates [eu-west-1]
```graphql
mutation setIsPortedFalseForBaseTemplates {
  documentation {
    setProgressReportTemplatesIsPortedFalse(
      input: {
        portingEntityType: BASE_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
      }
    )
  }
}
```

### Is ported false of all the generated templates [all regions]
```graphql
mutation setIsPortedFalseForGeneratedTemplates {
  documentation {
    setProgressReportTemplatesIsPortedFalse(
      input: {
        portingEntityType: GENERATED_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
      }
    )
  }
}
```

### Base template porting
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

### Map prompt set with default org_group [@chaitu]()
```sql
insert into cms.map_ai_prompt_set_org_groups (fk_prompt_set_id, fk_org_groups_id)
select aps.id, og.id from cms.ai_prompt_set as aps
                   join cms.org_groups as og
                   on og.type = 'DEFAULT'
where aps.id in ('202', '210', '211', '203')
returning *;
```

### Generated template porting
```graphql
mutation portTemplates {
  documentation {
    portProgressReportTemplates(
      input: {
        portingEntityType: GENERATED_TEMPLATES
        portingEntitySubType: PROGRESS_REPORT
        featureTypes: [DP_PROJECTS_REVAMP, MYP_PROJECTS_REVAMP, EDIT_CRITERIA_SET_LABEL]
        curriculumTypes: [IB_MYP, IB_DP, IB_PYP, UBD]
        shouldUpdateBaseTemplateIdMapping: false
      }
    )
  }
}
```

### Update parity labels of 'Final score', 'Local grade', 'Final grade'
- Because we want to mark the existing label in the template as edited labels so that
they are preserved when we update the label globally using the following script

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
 where cp.is_deleted = false;

foreach curriculum_program_id in array curriculum_program_ids loop
with etm_source as (
 select *
 from
 json_populate_recordset(
        null :: record,
 '[
  {"locale": "en", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637076325885, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "es", "content": "Calificación", "sub_type": "SCORE", "preset_id": 193614637080520190, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "fr", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637072131580, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "pt", "content": "Pontuação", "sub_type": "SCORE", "preset_id": 193614637072131579, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "tr", "content": "Puan", "sub_type": "SCORE", "preset_id": 193614637067937274, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "zh", "content": "分数", "sub_type": "SCORE", "preset_id": 193614637051160057, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614527495939249}, 
  {"locale": "en", "content": "Grade", "sub_type": "GRADE", "preset_id": 193614637273458179, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "es", "content": "Grado", "sub_type": "GRADE", "preset_id": 193614637273458180, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "fr", "content": "Année", "sub_type": "GRADE", "preset_id": 193614637273458178, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "pt", "content": "Série", "sub_type": "GRADE", "preset_id": 193614637273458177, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "tr", "content": "Sınıf seviyesi", "sub_type": "GRADE", "preset_id": 193614637273458176, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "zh", "content": "年级", "sub_type": "GRADE", "preset_id": 193614637269263871, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528628401330}, 
  {"locale": "en", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637273458185, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "es", "content": "Calificación", "sub_type": "SCORE", "preset_id": 193614637273458186, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "fr", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637273458184, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "pt", "content": "Pontuação", "sub_type": "SCORE", "preset_id": 193614637273458183, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "tr", "content": "Puan", "sub_type": "SCORE", "preset_id": 193614637273458182, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "zh", "content": "分数", "sub_type": "SCORE", "preset_id": 193614637273458181, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614528636789939}, 
  {"locale": "en", "content": "Grade", "sub_type": "GRADE", "preset_id": 193614637462201871, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "es", "content": "Grado", "sub_type": "GRADE", "preset_id": 193614637462201872, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "fr", "content": "Année", "sub_type": "GRADE", "preset_id": 193614637462201870, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "pt", "content": "Série", "sub_type": "GRADE", "preset_id": 193614637462201869, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "tr", "content": "Sınıf seviyesi", "sub_type": "GRADE", "preset_id": 193614637462201868, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "zh", "content": "年级", "sub_type": "GRADE", "preset_id": 193614637462201867, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614531996427444}, 
  {"locale": "en", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637462201877, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "es", "content": "Calificación", "sub_type": "SCORE", "preset_id": 193614637462201878, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "fr", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637462201876, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "pt", "content": "Pontuação", "sub_type": "SCORE", "preset_id": 193614637462201875, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "tr", "content": "Puan", "sub_type": "SCORE", "preset_id": 193614637462201874, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "zh", "content": "分数", "sub_type": "SCORE", "preset_id": 193614637462201873, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532004816053}, 
  {"locale": "en", "content": "Grade", "sub_type": "GRADE", "preset_id": 193614637650945563, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "es", "content": "Grado", "sub_type": "GRADE", "preset_id": 193614637650945564, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "fr", "content": "Année", "sub_type": "GRADE", "preset_id": 193614637650945562, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "pt", "content": "Série", "sub_type": "GRADE", "preset_id": 193614637650945561, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "tr", "content": "Sınıf seviyesi", "sub_type": "GRADE", "preset_id": 193614637650945560, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "zh", "content": "年级", "sub_type": "GRADE", "preset_id": 193614637650945559, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532457800886}, 
  {"locale": "en", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637650945569, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}, 
  {"locale": "es", "content": "Calificación", "sub_type": "SCORE", "preset_id": 193614637650945570, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}, 
  {"locale": "fr", "content": "Score", "sub_type": "SCORE", "preset_id": 193614637650945568, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}, 
  {"locale": "pt", "content": "Pontuação", "sub_type": "SCORE", "preset_id": 193614637650945567, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}, 
  {"locale": "tr", "content": "Puan", "sub_type": "SCORE", "preset_id": 193614637650945566, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}, 
  {"locale": "zh", "content": "分数", "sub_type": "SCORE", "preset_id": 193614637650945565, "criteria_type": "CATEGORY_PLACEHOLDER", "preset_set_id": 193614532466189495}]' :: json
      ) as x(
        preset_set_id bigint,
        preset_id bigint,
        locale text,
        content text,
        sub_type text,
        criteria_type criteria_type_enum
      )
 ),
  etm as (
   select acs.id as fk_entity_id,
    'ACADEMIC_CRITERIA_SET' as entity_type,
    'label' as entity_field_type,
    es.locale,
    es.content,
    acs.organization_id,
    es.preset_id
   from planner.academic_criteria_set acs
   join planner.curriculum_program_entity_map cpem on cpem.fk_entity_id = acs.id::text and cpem.entity_type = 'CRITERIA_SET'
   join etm_source es on es.criteria_type = acs.criteria_type and es.sub_type = acs.sub_type and (es.preset_set_id is null or es.preset_set_id = acs.preset_set_id)
   where cpem.fk_curriculum_program_id = curriculum_program_id
  )
 insert into platform.entity_translation_map(fk_entity_id, entity_type, entity_field_type, locale, content, organization_id, preset_id)
 select ne.fk_entity_id, ne.entity_type, ne.entity_field_type, ne.locale, ne.content, ne.organization_id, ne.preset_id
 from etm as ne
 on conflict (fk_entity_id, entity_type, entity_field_type, locale)
 do nothing ;


raise notice 'ported (%)', curriculum_program_id;
end loop;

raise notice 'porting done!';
end;

$$;
```

### Enable feature flags

Criteria label edit
- Enable FeatureFlag:ProgressReportEditableCriteriaSetLabels [14498@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14498)
- Enable FeatureFlag:TranscriptEditableCriteriaSetLabels [14499@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14499)

Projects v2
- Make FeatureFlag:EnableProjectGradebookInReports default true [14544@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14544)

One pager Transcript 
- Make FeatureFlag:ShowTranscriptsDownloadRevamp default true [14426@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14426)

Downloads update
- Make FeatureFlag:EnhancedProgressReportDownloads default true [14473@graphqlapi](https://github.com/toddle-edu/graphqlapi/pull/14473) 
 
