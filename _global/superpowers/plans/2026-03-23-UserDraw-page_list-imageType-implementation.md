# UserDraw page_list imageType Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an `imageType` request field to `POST /api/userDraw/page_list` so the endpoint returns base64 from `Result/<fileName>.png` when requested, while defaulting and falling back to `svg`.

**Architecture:** Keep the change scoped to the existing `page_list` read path. Normalize the requested image type in the system service layer, then route it into common-layer helpers that build the correct result file path and reuse the existing “file exists -> base64 / file missing -> null” behavior.

**Tech Stack:** Java 17, Spring Boot 3, Spring Data JPA, Maven multi-module build, JUnit 5, Mockito

---

## Preconditions

- Work from the repository root: `/mnt/f/IdeaProjects/goto-software`
- Prefer a dedicated branch or worktree before changing code
- Use the approved spec as the source of truth:
  `docs/superpowers/specs/2026-03-23-UserDraw-page_list-imageType-设计.md`
- The `goto-system` module sets `skipTests=true` in its POM, so every system-module test command in this plan must include `-DskipTests=false`
- The worktree is already dirty in unrelated areas, so every `git add` command must name exact files instead of using `git add .`

## File Map

### Existing files to modify

- `goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java:27-30`
  Add the new request-body field `imageType`
- `goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java:116-121`
  Normalize `criteria.imageType` and pass it into the image-fill helper
- `goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java:61-79`
  Stop hardcoding `.svg` inside `fillResultBase64ByUserDrawSimple1`
- `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java:984-1015`
  Add focused helpers for image-type normalization and user-draw result path construction

### New tests to create

- `goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java`
  Verify `png` path selection and null behavior when the requested file is missing
- `goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java`
  Verify `pageList` normalizes `imageType`, sets `userId`, and calls the common helper with the expected suffix

### Existing tests to extend

- `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java`
  Add focused tests for image-type normalization

## Implementation Notes

- Do not change `UserDrawController`; it already accepts `UserDrawQueryCriteria` as JSON body
- Do not change response shape; keep returning `image`
- Only `svg` and `png` are valid image types for this endpoint
- Normalize using this rule everywhere:

```java
if ("png".equalsIgnoreCase(imageType)) {
    return "png";
}
return "svg";
```

- Keep missing-file behavior non-fatal:

```java
if (Files.exists(Paths.get(filePath))) {
    return GFileUtils.imageToBase64(filePath);
}
return null;
```

## Task 1: Add Common-Layer Failing Tests

**Files:**
- Modify: `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java`
- Create: `goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java`

- [ ] **Step 1: Extend `PlotTypeUtilsTest` with normalization assertions**

Add tests like:

```java
@Test
void normalizeResultImageType_shouldAcceptPngCaseInsensitively() {
    assertEquals("png", PlotTypeUtils.normalizeResultImageType("PNG"));
}

@Test
void normalizeResultImageType_shouldFallbackToSvgForBlankOrInvalidValues() {
    assertEquals("svg", PlotTypeUtils.normalizeResultImageType(null));
    assertEquals("svg", PlotTypeUtils.normalizeResultImageType(""));
    assertEquals("svg", PlotTypeUtils.normalizeResultImageType("jpg"));
}
```

- [ ] **Step 2: Create `UserDrawCommonUtilsTest` that exercises suffix-based file lookup**

Use a temporary directory for `chartUser`, create a real `Result/plot.png`, and mock only the static param parsing calls:

```java
@ExtendWith(MockitoExtension.class)
class UserDrawCommonUtilsTest {

    @TempDir
    Path tempDir;

    @Mock
    private ComponentBean componentBean;

    @Mock
    private com.freedom.bean.Properties properties;

    @BeforeEach
    void setUp() {
        ConfigProperties configProperties = new ConfigProperties();
        RProperties rProperties = new RProperties();
        rProperties.setChartUser(tempDir.toString());
        configProperties.setR(rProperties);

        when(componentBean.getProperties()).thenReturn(properties);
        when(properties.getConfigProperties()).thenReturn(configProperties);
    }
}
```

Add two failing tests:

```java
@Test
void fillResultBase64ByUserDrawSimple1_shouldReadPngWhenRequested() throws Exception {
    UserDrawSimple1 row = new UserDrawSimple1();
    row.setTmpFolderName("tmp-1");
    row.setParamSettings(List.of(new ParamContainer()));

    Path resultDir = tempDir.resolve("tmp-1").resolve("Result");
    Files.createDirectories(resultDir);
    Files.write(
            resultDir.resolve("plot.png"),
            java.util.Base64.getDecoder().decode(
                    "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mP8/x8AAwMCAO+aF9sAAAAASUVORK5CYII="
            )
    );

    ObjectParam fileNameParam = new ObjectParam();
    fileNameParam.setParamName(PlotTypeUtils.FILE_NAME_FIELD_NAME);
    fileNameParam.setValue("plot");

    List<Param> params = List.of(fileNameParam);

    try (MockedStatic<ParamUtils> paramUtils = mockStatic(ParamUtils.class)) {
        paramUtils.when(() -> ParamUtils.paramContainerToParam(componentBean, row.getParamSettings()))
                .thenReturn(params);
        paramUtils.when(() -> ParamUtils.findByParamName(params, PlotTypeUtils.FILE_NAME_FIELD_NAME))
                .thenReturn(fileNameParam);

        UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, List.of(row), "png");
    }

    assertNotNull(row.getImage());
    assertNull(row.getParamSettings());
}

@Test
void fillResultBase64ByUserDrawSimple1_shouldReturnNullImageWhenRequestedPngIsMissing() {
    UserDrawSimple1 row = new UserDrawSimple1();
    row.setTmpFolderName("tmp-2");
    row.setParamSettings(List.of(new ParamContainer()));

    ObjectParam fileNameParam = new ObjectParam();
    fileNameParam.setParamName(PlotTypeUtils.FILE_NAME_FIELD_NAME);
    fileNameParam.setValue("plot");

    List<Param> params = List.of(fileNameParam);

    try (MockedStatic<ParamUtils> paramUtils = mockStatic(ParamUtils.class)) {
        paramUtils.when(() -> ParamUtils.paramContainerToParam(componentBean, row.getParamSettings()))
                .thenReturn(params);
        paramUtils.when(() -> ParamUtils.findByParamName(params, PlotTypeUtils.FILE_NAME_FIELD_NAME))
                .thenReturn(fileNameParam);

        UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, List.of(row), "png");
    }

    assertNull(row.getImage());
    assertNull(row.getParamSettings());
}
```

- [ ] **Step 3: Run the common-layer tests and confirm they fail for the right reason**

Run:

```bash
mvn -f goto/pom.xml -pl common -am -Dtest=PlotTypeUtilsTest,UserDrawCommonUtilsTest -DfailIfNoTests=false test
```

Expected:

- `PlotTypeUtilsTest` fails because `normalizeResultImageType` does not exist yet
- `UserDrawCommonUtilsTest` fails because `fillResultBase64ByUserDrawSimple1(..., "png")` does not exist yet

- [ ] **Step 4: Leave the worktree uncommitted while the tests are red**

Do not commit yet. Move directly to Task 2 so the common-layer test additions and implementation land together in one green commit.

## Task 2: Implement Common-Layer Helper Support

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java:984-1015`
- Modify: `goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java:61-79`
- Verify: `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java`
- Verify: `goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java`

- [ ] **Step 1: Add focused helper methods in `PlotTypeUtils`**

Implement these methods near the existing result-path helpers:

```java
public static String normalizeResultImageType(String imageType) {
    if ("png".equalsIgnoreCase(imageType)) {
        return "png";
    }
    return "svg";
}

public static String obtainUserDrawResultFilePath(
        ComponentBean componentBean,
        String tmpFolderName,
        String fileName,
        String fileSuffix
) {
    if (StringUtils.isAnyBlank(tmpFolderName, fileName, fileSuffix)) {
        return null;
    }
    return obtainUserDrawPath(componentBean, tmpFolderName) +
            File.separator +
            RESULT_FOLDER_NAME +
            File.separator +
            fileName +
            "." +
            fileSuffix;
}
```

- [ ] **Step 2: Overload `fillResultBase64ByUserDrawSimple1` to accept `imageType`**

Keep the existing no-arg variant for compatibility, but make it delegate:

```java
public static void fillResultBase64ByUserDrawSimple1(
        ComponentBean componentBean,
        List<UserDrawSimple1> userDrawSimple1s
) {
    fillResultBase64ByUserDrawSimple1(componentBean, userDrawSimple1s, "svg");
}

public static void fillResultBase64ByUserDrawSimple1(
        ComponentBean componentBean,
        List<UserDrawSimple1> userDrawSimple1s,
        String imageType
) {
    String normalizedImageType = PlotTypeUtils.normalizeResultImageType(imageType);
    if (GCollectionUtils.listNotNullNotEmpty(userDrawSimple1s)) {
        for (UserDrawSimple1 userDrawSimple1 : userDrawSimple1s) {
            List<Param> params = ParamUtils.paramContainerToParam(componentBean, userDrawSimple1.getParamSettings());
            Param param = ParamUtils.findByParamName(params, PlotTypeUtils.FILE_NAME_FIELD_NAME);

            if (param != null && param.getValue() != null) {
                String filePath = PlotTypeUtils.obtainUserDrawResultFilePath(
                        componentBean,
                        userDrawSimple1.getTmpFolderName(),
                        param.getValue().toString(),
                        normalizedImageType
                );
                userDrawSimple1.setImage(PlotTypeUtils.obtainPlotUserResultBase64(filePath));
            } else {
                userDrawSimple1.setImage(null);
            }

            userDrawSimple1.setParamSettings(null);
        }
    }
}
```

Use the new `PlotTypeUtils.obtainUserDrawResultFilePath(componentBean, tmpFolderName, fileName, normalizedImageType)` helper instead of manually concatenating `".svg"`.

- [ ] **Step 3: Re-run the common-layer tests and make sure they pass**

Run:

```bash
mvn -f goto/pom.xml -pl common -am -Dtest=PlotTypeUtilsTest,UserDrawCommonUtilsTest -DfailIfNoTests=false test
```

Expected:

- `PlotTypeUtilsTest` passes
- `UserDrawCommonUtilsTest` passes

- [ ] **Step 4: Commit the common-layer implementation**

```bash
git add goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java \
        goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java \
        goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java \
        goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java
git commit -m "feat: support imageType-aware UserDraw result lookup"
```

## Task 3: Add System-Layer Failing Tests for `pageList`

**Files:**
- Create: `goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java`

- [ ] **Step 1: Create a focused Mockito test for `UserDrawServiceImpl.pageList`**

Follow the local test style from `PlotServiceImplCancelStreamTest` and keep it constructor-based:

```java
@ExtendWith(MockitoExtension.class)
class UserDrawServiceImplPageListTest {

    @Mock
    private UserDrawRepository userDrawRepository;

    @Mock
    private UserDrawSimple1Repository userDrawSimple1Repository;

    @Mock
    private UserDrawMapper userDrawMapper;

    @Mock
    private ComponentBean componentBean;

    private UserDrawServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new UserDrawServiceImpl(
                userDrawRepository,
                userDrawSimple1Repository,
                userDrawMapper,
                componentBean
        );
    }
}
```

- [ ] **Step 2: Add the two key failing tests**

Use static mocks for `SecurityUtils` and `UserDrawCommonUtils` so the test only verifies service behavior:

```java
@Test
void pageList_shouldPassNormalizedPngImageTypeToImageFiller() {
    UserDrawQueryCriteria criteria = new UserDrawQueryCriteria();
    criteria.setImageType("PNG");

    List<UserDrawSimple1> rows = new ArrayList<>();
    Page<UserDrawSimple1> page = new PageImpl<>(rows);

    when(userDrawSimple1Repository.findAll(any(), any(Pageable.class))).thenReturn(page);

    try (MockedStatic<SecurityUtils> securityUtils = mockStatic(SecurityUtils.class);
         MockedStatic<UserDrawCommonUtils> userDrawCommonUtils = mockStatic(UserDrawCommonUtils.class)) {
        securityUtils.when(SecurityUtils::getCurrentUserId).thenReturn(100L);

        service.pageList(criteria);

        assertEquals(100L, criteria.getUserId());
        userDrawCommonUtils.verify(() ->
                UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, rows, "png"));
    }
}

@Test
void pageList_shouldFallbackToSvgWhenImageTypeIsMissing() {
    UserDrawQueryCriteria criteria = new UserDrawQueryCriteria();
    List<UserDrawSimple1> rows = new ArrayList<>();
    Page<UserDrawSimple1> page = new PageImpl<>(rows);

    when(userDrawSimple1Repository.findAll(any(), any(Pageable.class))).thenReturn(page);

    try (MockedStatic<SecurityUtils> securityUtils = mockStatic(SecurityUtils.class);
         MockedStatic<UserDrawCommonUtils> userDrawCommonUtils = mockStatic(UserDrawCommonUtils.class)) {
        securityUtils.when(SecurityUtils::getCurrentUserId).thenReturn(100L);

        service.pageList(criteria);

        userDrawCommonUtils.verify(() ->
                UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, rows, "svg"));
    }
}

@Test
void pageList_shouldFallbackToSvgWhenImageTypeIsInvalid() {
    UserDrawQueryCriteria criteria = new UserDrawQueryCriteria();
    criteria.setImageType("jpg");

    List<UserDrawSimple1> rows = new ArrayList<>();
    Page<UserDrawSimple1> page = new PageImpl<>(rows);

    when(userDrawSimple1Repository.findAll(any(), any(Pageable.class))).thenReturn(page);

    try (MockedStatic<SecurityUtils> securityUtils = mockStatic(SecurityUtils.class);
         MockedStatic<UserDrawCommonUtils> userDrawCommonUtils = mockStatic(UserDrawCommonUtils.class)) {
        securityUtils.when(SecurityUtils::getCurrentUserId).thenReturn(100L);

        service.pageList(criteria);

        userDrawCommonUtils.verify(() ->
                UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, rows, "svg"));
    }
}
```

- [ ] **Step 3: Run the system-layer test and confirm it fails**

Run:

```bash
mvn -f goto/pom.xml -pl system -am -DskipTests=false -Dtest=UserDrawServiceImplPageListTest -DfailIfNoTests=false test
```

Expected:

- the test fails because `UserDrawQueryCriteria` has no `imageType`
- or the test fails because `UserDrawServiceImpl.pageList` still calls the old two-argument helper

- [ ] **Step 4: Leave the worktree uncommitted while the tests are red**

Do not commit yet. Move directly to Task 4 so the system-layer test and implementation land together in one green commit.

## Task 4: Implement `pageList` Request and Propagation Changes

**Files:**
- Modify: `goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java:27-30`
- Modify: `goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java:116-121`
- Verify: `goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java`

- [ ] **Step 1: Add `imageType` to `UserDrawQueryCriteria`**

Make the DTO change minimal:

```java
@Data
public class UserDrawQueryCriteria extends BaseQueryCriteria {
    @Query(type = Query.Type.EQUAL)
    private Long userId;

    private String imageType;
}
```

- [ ] **Step 2: Normalize and pass `imageType` in `UserDrawServiceImpl.pageList`**

Update the method to keep the existing query flow unchanged:

```java
@Override
public CommonResponse pageList(UserDrawQueryCriteria criteria) {
    criteria.setUserId(SecurityUtils.getCurrentUserId());
    String imageType = PlotTypeUtils.normalizeResultImageType(criteria.getImageType());
    Page<UserDrawSimple1> page = userDrawSimple1Repository.findAll(
            (root, criteriaQuery, criteriaBuilder) -> QueryHelp.getPredicate(root, criteria, criteriaBuilder),
            criteria.obtainPageable()
    );
    UserDrawCommonUtils.fillResultBase64ByUserDrawSimple1(componentBean, page.getContent(), imageType);
    return ResponseUtils.successData(PageUtil.toPage(page));
}
```

Do not mutate the query predicate set beyond `userId`; `imageType` is not a database filter.

- [ ] **Step 3: Run the system-layer test and make sure it passes**

Run:

```bash
mvn -f goto/pom.xml -pl system -am -DskipTests=false -Dtest=UserDrawServiceImplPageListTest -DfailIfNoTests=false test
```

Expected:

- `UserDrawServiceImplPageListTest` passes

- [ ] **Step 4: Commit the system-layer implementation**

```bash
git add goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java \
        goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java \
        goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java
git commit -m "feat: allow pageList imageType selection"
```

## Task 5: Run Cross-Module Regression Checks

**Files:**
- Verify: `goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java`
- Verify: `goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java`
- Verify: `goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java`
- Review: `goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java`
- Review: `goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java`
- Review: `goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java`
- Review: `goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java`

- [ ] **Step 1: Run the common-layer tests again from a clean compile state**

Run:

```bash
mvn -f goto/pom.xml -pl common -am -Dtest=PlotTypeUtilsTest,UserDrawCommonUtilsTest -DfailIfNoTests=false test
```

Expected:

- both common-module test classes pass

- [ ] **Step 2: Run the system-layer tests again from a clean compile state**

Run:

```bash
mvn -f goto/pom.xml -pl system -am -DskipTests=false -Dtest=UserDrawServiceImplPageListTest -DfailIfNoTests=false test
```

Expected:

- the system-module test class passes

- [ ] **Step 3: Review the final diff for accidental scope creep**

Run:

```bash
git status --short
git diff -- goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java \
             goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java \
             goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java \
             goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java \
             goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java \
             goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java \
             goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java
```

Confirm:

- only the approved files changed
- no controller, download, or analysis-generation code was touched
- default `svg` behavior is preserved

- [ ] **Step 4: If the review step required any cleanup, make a final commit**

Use this only if Task 5 Step 3 caused code changes:

```bash
git add goto/common/src/main/java/com/freedom/util/PlotTypeUtils.java \
        goto/common/src/main/java/com/freedom/util/UserDrawCommonUtils.java \
        goto/system/src/main/java/com/freedom/service/dto/UserDrawQueryCriteria.java \
        goto/system/src/main/java/com/freedom/service/impl/UserDrawServiceImpl.java \
        goto/common/src/test/java/com/freedom/util/PlotTypeUtilsTest.java \
        goto/common/src/test/java/com/freedom/util/UserDrawCommonUtilsTest.java \
        goto/system/src/test/java/com/freedom/service/impl/UserDrawServiceImplPageListTest.java
git commit -m "test: verify pageList imageType regression coverage"
```

If no cleanup was needed, skip this step.
