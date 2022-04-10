```java
@SpringBootTest
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
public class ParameterValidationUtilsTest {
    
    @InjectMocks
    ParameterValidationUtils mockParameterValidationUtils;
    
    @Mock
    InternationalMilesPostRegistrationProperties mockInternationalMilesPostRegistrationProperties;
    
    @Mock
    JmbCoreLogger mockJmbCoreLogger;
    
    @BeforeEach
    void setUp() {
        when(mockInternationalMilesPostRegistrationProperties.getProperty(CoreConstants.JMBCMNE001)).thenReturn("The request parameter is invalid. ");
        when(mockInternationalMilesPostRegistrationProperties.getProperty(InternationalMilesPostRegistrationConstant.DATE_FORMAT)).thenReturn("uuuuMMdd");
    }
    
    @ParameterizedTest
    @MethodSource("getDateSources")
    void validateActivityDateInvalidDateTest(String invalidDate) {
        // throws customException if the date is invalid
        CustomException customException = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, invalidDate));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException.getMessage());
    }
    
    static Stream<String> getDateSources() {
        return Stream.of("20210229", "20210431", "20210532");
    }
    
    @Test
    void validateActivityDateConditionsTest() {
        LocalDate now = LocalDate.now();
        String dateFormat = "yyyyMMdd";
        
        // throws customException if get a future date 
        String futureDateStr = now.plusDays(1).toString(dateFormat);
        CustomException customException1 = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, futureDateStr));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException1.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException1.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(true, now.toString(dateFormat)));
        
        // throws customException if it is JAL group and the date is older than 6 months from current date
        String sixMonthsAgo = now.plusMonths(-6).plusDays(-1).toString(dateFormat);
        CustomException customException2 = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, sixMonthsAgo));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException2.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException2.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(true, now.plusMonths(-6).toString(dateFormat)));
        
        // throws customException if it is partnership group and the date is older than 6 months plus 14 days
        String sixMonthsPlus14DaysAgo = now.plusMonths(-6).plusDays(-15).toString(dateFormat);
        CustomException customException3= Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(false, sixMonthsPlus14DaysAgo));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException3.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException3.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(false, now.plusMonths(-6).plusDays(-14).toString(dateFormat)));
    }
    
    @ParameterizedTest
    @MethodSource("getValidCarrierCodeResources")
    void validateCarrierCodePositiveTest(String carrierCode) {
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateCarrierCode(carrierCode));
    }
    
    static Stream<String> getValidCarrierCodeResources() {
        return Stream.of("JAL", "GKF");
    }
    
    @ParameterizedTest
    @MethodSource("getInvalidCarrierCodeResources")
    void validateCarrierCodeNegativeTest(String carrierCode) {
        CustomException customException = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateCarrierCode(carrierCode));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException.getMessage());
    }
    
    static Stream<String> getInvalidCarrierCodeResources() {
        return Stream.of(null, "", " ", "  ");
    }
```

