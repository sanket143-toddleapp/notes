# Student report template optimization

```
basicReportDetails: 1.083s

Okayish, most of the times it is in millis.

attendanceRecordingTypeForProgressReport: 2.234s
getCurriculumReportConfig: 6.269s
_getAttendanceOptionsForReport: 8.502s
curriculumConfig, attendanceOptions, attendanceRecordingType: 8.509s

_getAttendanceOptionsForReport is something that can be optimised, need to connect with Pawan.
getCurriculumReportConfig -> configurations are fetched in this, anything major can't be done currently.
attendanceRecordingTypeForProgressReport -> already takes the least time in these three parallely, so least concern rn.

templateDataArray, studentSubjects: 1.271s

Okayish.


promises: 2.788s

fetching academic criteria sets takes time in this. The query is just too heavy currently, can be revisited.

Something 2: 0.452ms
hasGradeBoundariesForCategories: 0.014ms
_addLocalGrades: 0.497ms

_addSubjectIconUrls: 2.686s

Need to check this -> maybe a second can be optimised.

fetchAcademicYearsById: 2.575s
settings, gradeWiseSettings: 5.821s
attendanceRecordingTypeForProgressReport: 1.731s         --> in orgConfig
Multiple in orgConfig: 16.246s
addOrgConfigurationsForProgressReportTemplate: 22.592s
fetchCategoryAndSubjectWiseFinalScoresByStudentProgressReportIds: 28.838s
largePromises: 28.839s

Three calls in large promises:
    fetchCategoryAndSubjectWiseFinalScoresByStudentProgressReportIds -> 
    addOrgConfigurationsForProgressReportTemplate
    fetchAcademicYearsById


_filterAcademicCriteriaSetsForProgressReportTemplate: 4.356ms
syncCustomFieldsForStudentInformation: 2.253s
15200: 6.261ms
newTemplate: 29.419ms
_generateStudentProgressReportTaskTemplate: 30.748ms
generateStudentProgressReportTemplateDynamicNodes: 1.222s
generateSignatoriesNodes: 0.314ms
```


