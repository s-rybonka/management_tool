# Indicators Feature Development Plan

## Architecture Approach

### Design Patterns and Principles

1. **Repository Pattern**
   - Why: To abstract data access logic and provide a clean API for indicator operations
   - Benefits: 
     - Centralizes data access logic
     - Makes testing easier by allowing mock repositories
     - Provides flexibility to change data sources without affecting business logic
   - Implementation: Create `IndicatorRepository` and `IndicatorValueRepository` classes in `repositories.py`

2. **Factory Pattern**
   - Why: For creating different types of indicators (quantitative/qualitative)
   - Benefits:
     - Encapsulates indicator creation logic
     - Makes it easy to add new indicator types
     - Provides consistent interface for indicator creation
   - Implementation: Create `IndicatorFactory` class in `factories.py`

3. **Strategy Pattern**
   - Why: For different calculation methods (sum, average, percentage)
   - Benefits:
     - Makes it easy to add new calculation methods
     - Keeps calculation logic separate from indicator logic
     - Allows runtime selection of calculation method
   - Implementation: Create `CalculationStrategy` interface and concrete strategies in `strategies.py`

4. **Observer Pattern**
   - Why: For real-time updates when indicator values change
   - Benefits:
     - Decouples indicator updates from UI updates
     - Enables real-time dashboard updates
     - Makes it easy to add new observers (e.g., notifications)
   - Implementation: Create `IndicatorObserver` interface and concrete observers in `observers.py`

### SOLID Principles

1. **Single Responsibility Principle**
   - Each class has one reason to change
   - Example: Separate classes for indicator creation, calculation, and validation
   - Implementation: Create separate service classes for each responsibility

2. **Open/Closed Principle**
   - Open for extension, closed for modification
   - Example: Indicator types can be extended without modifying existing code
   - Implementation: Use abstract base classes and interfaces

3. **Liskov Substitution Principle**
   - Subtypes must be substitutable for their base types
   - Example: All indicator types must implement the same interface
   - Implementation: Create `BaseIndicator` abstract class

4. **Interface Segregation Principle**
   - Clients shouldn't depend on interfaces they don't use
   - Example: Separate interfaces for indicator creation and calculation
   - Implementation: Create specific interfaces for each operation

5. **Dependency Inversion Principle**
   - Depend on abstractions, not concretions
   - Example: Use interfaces for repositories and services
   - Implementation: Use dependency injection in service classes

## Django App Structure

```
indicators/
├── migrations/
│   └── __init__.py
├── tests/
│   ├── conftest.py
│   ├── factories.py
│   ├── test_views.py
│   ├── test_models.py
│   └── test_serializers.py
├── admin.py
├── apps.py
├── constants.py
├── models.py
├── serializers.py
├── permissions.py
├── querysets.py
├── filters.py
├── validators.py
├── views.py
├── urls.py
├── repositories.py
├── services.py
├── factories.py
├── strategies.py
└── observers.py
```

## Development Tasks

### Backend Tasks

#### 1. Core Models Setup

- **Task**: Create Quantitative and Qualitative Indicator Models in `models.py`
  - Create `BaseIndicator` abstract model:
    ```python
    class BaseIndicator(models.Model):
        class Meta:
            abstract = True
        
        name = models.CharField(max_length=255)
        description = models.TextField()
        reporting_frequency = models.CharField(max_length=50)
        disaggregation_fields = models.JSONField(default=dict)
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
    ```

  - Create `QuantitativeIndicator` model:
    ```python
    class QuantitativeIndicator(BaseIndicator):
        unit_of_measurement = models.CharField(max_length=50)
        baseline_value = models.DecimalField(max_digits=15, decimal_places=2)
        target_value = models.DecimalField(max_digits=15, decimal_places=2)
        calculation_method = models.ForeignKey('CalculationMethod', on_delete=models.SET_NULL, null=True)
        unicity_field = models.CharField(max_length=255, null=True, blank=True)
    ```

  - Create `QualitativeIndicator` model:
    ```python
    class QualitativeIndicator(BaseIndicator):
        rating_scale = models.JSONField(default=list)
        narrative_template = models.TextField(null=True, blank=True)
        assessment_criteria = models.JSONField(default=list)
    ```

  - Add proper validators in `validators.py`
  - Add model methods for common operations
  - Tips: Use Django's model fields appropriately, add proper validators
  - Acceptance Criteria:
    1. Models can be created and saved successfully
    2. All fields are properly validated
    3. Model methods work as expected
    4. Database migrations are created and applied successfully
    5. Abstract model prevents direct instantiation
    6. Proper inheritance is maintained

- **Task**: Create IndicatorValue and DisaggregatedValue Models in `models.py`
  - Create `IndicatorValue` model:
    ```python
    class IndicatorValue(models.Model):
        VALUE_TYPES = (
            ('BASELINE', 'Baseline'),
            ('TARGET', 'Target'),
            ('CURRENT', 'Current')
        )
        indicator = models.ForeignKey('BaseIndicator', on_delete=models.CASCADE)
        value = models.JSONField()  # Stores both numeric and qualitative values
        date = models.DateField()
        source = models.CharField(max_length=255)
        value_type = models.CharField(max_length=20, choices=VALUE_TYPES)
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
    ```

  - Create `DisaggregatedValue` model:
    ```python
    class DisaggregatedValue(models.Model):
        indicator_value = models.ForeignKey(IndicatorValue, on_delete=models.CASCADE)
        category = models.CharField(max_length=100)  # e.g., 'gender', 'age_group'
        subcategory = models.CharField(max_length=100)  # e.g., 'male', '18-25'
        value = models.JSONField()
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
    ```

  - Add proper validators in `validators.py`
  - Add model methods for value operations
  - Tips: Use appropriate field types for different value types
  - Acceptance Criteria:
    1. Models can be created and saved successfully
    2. Values are properly validated
    3. Disaggregated values are stored correctly
    4. Database migrations are created and applied successfully
    5. Value types are properly enforced
    6. Historical tracking is maintained

- **Task**: Create CalculationMethod and Formula Models in `models.py`
  - Create `CalculationMethod` model:
    ```python
    class CalculationMethod(models.Model):
        METHOD_TYPES = (
            ('SUM', 'Sum'),
            ('AVERAGE', 'Average'),
            ('PERCENTAGE', 'Percentage'),
            ('COUNT', 'Count'),
            ('COUNT_UNIQUE', 'Count Unique'),
            ('CUSTOM', 'Custom')
        )
        name = models.CharField(max_length=255)
        type = models.CharField(max_length=20, choices=METHOD_TYPES)
        description = models.TextField()
        parameters = models.JSONField(default=dict)
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
    ```

  - Create `Formula` model:
    ```python
    class Formula(models.Model):
        calculation_method = models.ForeignKey(CalculationMethod, on_delete=models.CASCADE)
        expression = models.TextField()  # Stores mathematical expression
        variables = models.JSONField(default=list)  # List of variable names
        conditions = models.JSONField(default=list)  # List of conditional filters
        created_at = models.DateTimeField(auto_now_add=True)
        updated_at = models.DateTimeField(auto_now=True)
    ```

  - Add proper validators in `validators.py`
  - Add model methods for calculations
  - Tips: Use choices field for method types
  - Acceptance Criteria:
    1. Models can be created and saved successfully
    2. Calculation methods are properly validated
    3. Formulas are stored and evaluated correctly
    4. Database migrations are created and applied successfully
    5. Custom calculations are supported
    6. Parameter validation works correctly

#### 2. Data Access Layer

- **Task**: Create Comprehensive Repository Classes in `repositories.py`
  - Create `BaseIndicatorRepository` abstract class:
    ```python
    class BaseIndicatorRepository(ABC):
        @abstractmethod
        def get_by_id(self, id: int) -> BaseIndicator:
            pass
        
        @abstractmethod
        def get_all(self) -> QuerySet[BaseIndicator]:
            pass
        
        @abstractmethod
        def create(self, data: dict) -> BaseIndicator:
            pass
        
        @abstractmethod
        def update(self, id: int, data: dict) -> BaseIndicator:
            pass
        
        @abstractmethod
        def delete(self, id: int) -> bool:
            pass
        
        @abstractmethod
        def get_by_type(self, indicator_type: str) -> QuerySet[BaseIndicator]:
            pass
    ```

  - Create `QuantitativeIndicatorRepository` class:
    ```python
    class QuantitativeIndicatorRepository(BaseIndicatorRepository):
        def get_by_calculation_method(self, method_id: int) -> QuerySet[QuantitativeIndicator]:
            pass
        
        def get_with_targets(self) -> QuerySet[QuantitativeIndicator]:
            pass
        
        def get_with_baselines(self) -> QuerySet[QuantitativeIndicator]:
            pass
    ```

  - Create `QualitativeIndicatorRepository` class:
    ```python
    class QualitativeIndicatorRepository(BaseIndicatorRepository):
        def get_by_rating_scale(self, scale_type: str) -> QuerySet[QualitativeIndicator]:
            pass
        
        def get_with_narrative(self) -> QuerySet[QualitativeIndicator]:
            pass
    ```

  - Create `IndicatorValueRepository` class:
    ```python
    class IndicatorValueRepository:
        def get_by_indicator(self, indicator_id: int) -> QuerySet[IndicatorValue]:
            pass
        
        def get_by_date_range(self, start_date: date, end_date: date) -> QuerySet[IndicatorValue]:
            pass
        
        def get_by_value_type(self, value_type: str) -> QuerySet[IndicatorValue]:
            pass
        
        def get_latest_values(self) -> QuerySet[IndicatorValue]:
            pass
    ```

  - Create `DisaggregatedValueRepository` class:
    ```python
    class DisaggregatedValueRepository:
        def get_by_category(self, category: str) -> QuerySet[DisaggregatedValue]:
            pass
        
        def get_by_subcategory(self, category: str, subcategory: str) -> QuerySet[DisaggregatedValue]:
            pass
        
        def get_aggregated_values(self, indicator_value_id: int) -> dict:
            pass
    ```

  - Tips: Implement proper error handling and logging
  - Acceptance Criteria:
    1. All repository methods work correctly
    2. Error handling is implemented properly
    3. Logging is implemented for all operations
    4. Methods return correct types
    5. Abstract base class prevents direct instantiation
    6. Type hints are properly used
    7. Query optimization is implemented

#### 3. Business Logic Layer

- **Task**: Create Comprehensive Service Classes in `services.py`
  - Create `BaseIndicatorService` abstract class:
    ```python
    class BaseIndicatorService(ABC):
        def __init__(self, repository: BaseIndicatorRepository):
            self.repository = repository
        
        @abstractmethod
        def create_indicator(self, data: dict) -> BaseIndicator:
            pass
        
        @abstractmethod
        def update_indicator(self, id: int, data: dict) -> BaseIndicator:
            pass
        
        @abstractmethod
        def delete_indicator(self, id: int) -> bool:
            pass
        
        @abstractmethod
        def validate_indicator(self, data: dict) -> bool:
            pass
    ```

  - Create `QuantitativeIndicatorService` class:
    ```python
    class QuantitativeIndicatorService(BaseIndicatorService):
        def calculate_value(self, indicator_id: int, data: dict) -> Decimal:
            pass
        
        def validate_calculation(self, indicator_id: int, value: Decimal) -> bool:
            pass
        
        def get_progress(self, indicator_id: int) -> dict:
            pass
        
        def get_trend(self, indicator_id: int, period: str) -> list:
            pass
    ```

  - Create `QualitativeIndicatorService` class:
    ```python
    class QualitativeIndicatorService(BaseIndicatorService):
        def evaluate_rating(self, indicator_id: int, data: dict) -> dict:
            pass
        
        def generate_narrative(self, indicator_id: int, data: dict) -> str:
            pass
        
        def validate_assessment(self, indicator_id: int, data: dict) -> bool:
            pass
    ```

  - Create `IndicatorValueService` class:
    ```python
    class IndicatorValueService:
        def create_value(self, indicator_id: int, data: dict) -> IndicatorValue:
            pass
        
        def update_value(self, value_id: int, data: dict) -> IndicatorValue:
            pass
        
        def delete_value(self, value_id: int) -> bool:
            pass
        
        def get_historical_values(self, indicator_id: int) -> list:
            pass
        
        def calculate_aggregates(self, indicator_id: int) -> dict:
            pass
    ```

  - Create `DisaggregationService` class:
    ```python
    class DisaggregationService:
        def create_disaggregated_value(self, value_id: int, data: dict) -> DisaggregatedValue:
            pass
        
        def update_disaggregated_value(self, id: int, data: dict) -> DisaggregatedValue:
            pass
        
        def delete_disaggregated_value(self, id: int) -> bool:
            pass
        
        def get_disaggregated_summary(self, indicator_id: int) -> dict:
            pass
        
        def validate_disaggregation(self, indicator_id: int, data: dict) -> bool:
            pass
    ```

  - Tips: Implement proper error handling and logging
  - Acceptance Criteria:
    1. All service methods work correctly
    2. Business logic is properly implemented
    3. Error handling is implemented properly
    4. Services are properly tested
    5. Abstract base class prevents direct instantiation
    6. Type hints are properly used
    7. Business rules are properly enforced

#### 4. API Layer

- **Task**: Create Comprehensive Serializers in `serializers.py`
  - Create `BaseIndicatorSerializer` abstract class:
    ```python
    class BaseIndicatorSerializer(serializers.ModelSerializer):
        class Meta:
            abstract = True
        
        def validate(self, data):
            # Common validation logic
            return data
        
        def to_representation(self, instance):
            # Common representation logic
            return super().to_representation(instance)
    ```

  - Create `QuantitativeIndicatorSerializer` class:
    ```python
    class QuantitativeIndicatorSerializer(BaseIndicatorSerializer):
        calculation_method = CalculationMethodSerializer(read_only=True)
        current_value = serializers.SerializerMethodField()
        progress = serializers.SerializerMethodField()
        
        class Meta:
            model = QuantitativeIndicator
            fields = ['id', 'name', 'description', 'unit_of_measurement', 
                     'baseline_value', 'target_value', 'calculation_method',
                     'current_value', 'progress', 'reporting_frequency']
        
        def get_current_value(self, obj):
            return obj.get_latest_value()
        
        def get_progress(self, obj):
            return obj.calculate_progress()
    ```

  - Create `QualitativeIndicatorSerializer` class:
    ```python
    class QualitativeIndicatorSerializer(BaseIndicatorSerializer):
        latest_assessment = serializers.SerializerMethodField()
        rating_summary = serializers.SerializerMethodField()
        
        class Meta:
            model = QualitativeIndicator
            fields = ['id', 'name', 'description', 'rating_scale',
                     'narrative_template', 'latest_assessment',
                     'rating_summary', 'reporting_frequency']
        
        def get_latest_assessment(self, obj):
            return obj.get_latest_assessment()
        
        def get_rating_summary(self, obj):
            return obj.get_rating_summary()
    ```

  - Create `IndicatorValueSerializer` class:
    ```python
    class IndicatorValueSerializer(serializers.ModelSerializer):
        disaggregated_values = DisaggregatedValueSerializer(many=True, read_only=True)
        indicator = serializers.PrimaryKeyRelatedField(read_only=True)
        
        class Meta:
            model = IndicatorValue
            fields = ['id', 'indicator', 'value', 'date', 'source',
                     'value_type', 'disaggregated_values']
        
        def validate_value(self, value):
            indicator = self.context['indicator']
            if isinstance(indicator, QuantitativeIndicator):
                if not isinstance(value, (int, float)):
                    raise serializers.ValidationError("Value must be numeric")
            return value
    ```

  - Create `DisaggregatedValueSerializer` class:
    ```python
    class DisaggregatedValueSerializer(serializers.ModelSerializer):
        class Meta:
            model = DisaggregatedValue
            fields = ['id', 'indicator_value', 'category', 'subcategory',
                     'value']
        
        def validate(self, data):
            indicator_value = data['indicator_value']
            if not indicator_value.indicator.disaggregation_fields.get(data['category']):
                raise serializers.ValidationError("Invalid category")
            return data
    ```

  - Tips: Implement proper validation and error handling
  - Acceptance Criteria:
    1. All serializers properly validate data
    2. Nested serialization works correctly
    3. Custom validation methods are implemented
    4. Error messages are clear and helpful
    5. Serializers handle both read and write operations
    6. Type conversion is handled properly
    7. Performance is optimized for large datasets

- **Task**: Create Comprehensive ViewSets in `views.py`
  - Create `BaseIndicatorViewSet` abstract class:
    ```python
    class BaseIndicatorViewSet(viewsets.ModelViewSet):
        permission_classes = [IsAuthenticated, HasIndicatorPermission]
        pagination_class = StandardResultsSetPagination
        
        def get_queryset(self):
            return self.queryset.filter(
                organization=self.request.user.organization
            )
        
        def perform_create(self, serializer):
            serializer.save(organization=self.request.user.organization)
    ```

  - Create `QuantitativeIndicatorViewSet` class:
    ```python
    class QuantitativeIndicatorViewSet(BaseIndicatorViewSet):
        serializer_class = QuantitativeIndicatorSerializer
        queryset = QuantitativeIndicator.objects.all()
        
        @action(detail=True, methods=['get'])
        def calculate_value(self, request, pk=None):
            indicator = self.get_object()
            data = request.data
            value = indicator.calculate_value(data)
            return Response({'value': value})
        
        @action(detail=True, methods=['get'])
        def trend(self, request, pk=None):
            indicator = self.get_object()
            period = request.query_params.get('period', 'monthly')
            trend_data = indicator.get_trend(period)
            return Response(trend_data)
    ```

  - Create `QualitativeIndicatorViewSet` class:
    ```python
    class QualitativeIndicatorViewSet(BaseIndicatorViewSet):
        serializer_class = QualitativeIndicatorSerializer
        queryset = QualitativeIndicator.objects.all()
        
        @action(detail=True, methods=['post'])
        def assess(self, request, pk=None):
            indicator = self.get_object()
            assessment = indicator.evaluate_rating(request.data)
            return Response(assessment)
        
        @action(detail=True, methods=['get'])
        def history(self, request, pk=None):
            indicator = self.get_object()
            history = indicator.get_assessment_history()
            return Response(history)
    ```

  - Create `IndicatorValueViewSet` class:
    ```python
    class IndicatorValueViewSet(viewsets.ModelViewSet):
        serializer_class = IndicatorValueSerializer
        permission_classes = [IsAuthenticated, HasValuePermission]
        queryset = IndicatorValue.objects.all()
        
        def get_queryset(self):
            indicator_id = self.request.query_params.get('indicator')
            if indicator_id:
                return self.queryset.filter(indicator_id=indicator_id)
            return self.queryset
        
        @action(detail=True, methods=['post'])
        def disaggregate(self, request, pk=None):
            value = self.get_object()
            data = request.data
            disaggregated = value.create_disaggregated_values(data)
            return Response(disaggregated)
    ```

  - Create `DisaggregatedValueViewSet` class:
    ```python
    class DisaggregatedValueViewSet(viewsets.ModelViewSet):
        serializer_class = DisaggregatedValueSerializer
        permission_classes = [IsAuthenticated, HasDisaggregationPermission]
        queryset = DisaggregatedValue.objects.all()
        
        def get_queryset(self):
            value_id = self.request.query_params.get('value')
            if value_id:
                return self.queryset.filter(indicator_value_id=value_id)
            return self.queryset
        
        @action(detail=False, methods=['get'])
        def summary(self, request):
            category = request.query_params.get('category')
            summary = self.get_queryset().get_summary(category)
            return Response(summary)
    ```

  - Tips: Implement proper permission checks and error handling
  - Acceptance Criteria:
    1. All viewsets handle CRUD operations correctly
    2. Custom actions work as expected
    3. Permissions are properly enforced
    4. Query parameters are properly handled
    5. Pagination works correctly
    6. Error responses are properly formatted
    7. Performance is optimized for large datasets

#### 5. Frontend Tasks

- **Task**: Create Indicator Management Components
  - Create `IndicatorList` component:
    ```typescript
    interface IndicatorListProps {
      organizationId: string;
      onIndicatorSelect: (indicator: Indicator) => void;
    }
    
    const IndicatorList: React.FC<IndicatorListProps> = ({
      organizationId,
      onIndicatorSelect,
    }) => {
      const [indicators, setIndicators] = useState<Indicator[]>([]);
      const [loading, setLoading] = useState(true);
      const [error, setError] = useState<string | null>(null);
      
      useEffect(() => {
        fetchIndicators();
      }, [organizationId]);
      
      const fetchIndicators = async () => {
        try {
          const response = await api.get(`/indicators/?organization=${organizationId}`);
          setIndicators(response.data);
        } catch (err) {
          setError('Failed to fetch indicators');
        } finally {
          setLoading(false);
        }
      };
      
      if (loading) return <LoadingSpinner />;
      if (error) return <ErrorMessage message={error} />;
      
      return (
        <div className="indicator-list">
          <h2>Indicators</h2>
          <div className="filters">
            <TypeFilter />
            <SearchFilter />
            <SortOptions />
          </div>
          <div className="indicators-grid">
            {indicators.map((indicator) => (
              <IndicatorCard
                key={indicator.id}
                indicator={indicator}
                onClick={() => onIndicatorSelect(indicator)}
              />
            ))}
          </div>
          <Pagination
            total={indicators.length}
            pageSize={10}
            onChange={handlePageChange}
          />
        </div>
      );
    };
    ```

  - Create `IndicatorForm` component:
    ```typescript
    interface IndicatorFormProps {
      initialData?: Indicator;
      onSubmit: (data: IndicatorFormData) => Promise<void>;
    }
    
    const IndicatorForm: React.FC<IndicatorFormProps> = ({
      initialData,
      onSubmit,
    }) => {
      const [formData, setFormData] = useState<IndicatorFormData>(
        initialData || defaultFormData
      );
      const [errors, setErrors] = useState<FormErrors>({});
      const [submitting, setSubmitting] = useState(false);
      
      const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        setSubmitting(true);
        try {
          await onSubmit(formData);
        } catch (err) {
          setErrors(parseFormErrors(err));
        } finally {
          setSubmitting(false);
        }
      };
      
      return (
        <form onSubmit={handleSubmit} className="indicator-form">
          <FormSection title="Basic Information">
            <TextField
              label="Name"
              value={formData.name}
              onChange={(e) => setFormData({ ...formData, name: e.target.value })}
              error={errors.name}
            />
            <TextArea
              label="Description"
              value={formData.description}
              onChange={(e) =>
                setFormData({ ...formData, description: e.target.value })
              }
              error={errors.description}
            />
          </FormSection>
          
          <FormSection title="Type and Measurement">
            <TypeSelector
              value={formData.type}
              onChange={(type) => setFormData({ ...formData, type })}
            />
            {formData.type === 'QUANTITATIVE' && (
              <QuantitativeFields
                data={formData}
                onChange={setFormData}
                errors={errors}
              />
            )}
            {formData.type === 'QUALITATIVE' && (
              <QualitativeFields
                data={formData}
                onChange={setFormData}
                errors={errors}
              />
            )}
          </FormSection>
          
          <FormSection title="Reporting">
            <FrequencySelector
              value={formData.reporting_frequency}
              onChange={(frequency) =>
                setFormData({ ...formData, reporting_frequency: frequency })
              }
            />
            <DisaggregationFields
              value={formData.disaggregation_fields}
              onChange={(fields) =>
                setFormData({ ...formData, disaggregation_fields: fields })
              }
            />
          </FormSection>
          
          <FormActions>
            <Button type="submit" loading={submitting}>
              {initialData ? 'Update' : 'Create'} Indicator
            </Button>
          </FormActions>
        </form>
      );
    };
    ```

  - Create `IndicatorDetail` component:
    ```typescript
    interface IndicatorDetailProps {
      indicator: Indicator;
      onUpdate: (data: Partial<Indicator>) => Promise<void>;
    }
    
    const IndicatorDetail: React.FC<IndicatorDetailProps> = ({
      indicator,
      onUpdate,
    }) => {
      const [activeTab, setActiveTab] = useState('overview');
      const [values, setValues] = useState<IndicatorValue[]>([]);
      const [loading, setLoading] = useState(true);
      
      useEffect(() => {
        fetchValues();
      }, [indicator.id]);
      
      const fetchValues = async () => {
        try {
          const response = await api.get(
            `/indicators/${indicator.id}/values/`
          );
          setValues(response.data);
        } catch (err) {
          // Handle error
        } finally {
          setLoading(false);
        }
      };
      
      return (
        <div className="indicator-detail">
          <Header>
            <h1>{indicator.name}</h1>
            <IndicatorActions
              indicator={indicator}
              onUpdate={onUpdate}
            />
          </Header>
          
          <Tabs
            activeTab={activeTab}
            onChange={setActiveTab}
            tabs={[
              { id: 'overview', label: 'Overview' },
              { id: 'values', label: 'Values' },
              { id: 'disaggregation', label: 'Disaggregation' },
              { id: 'history', label: 'History' },
            ]}
          />
          
          <TabContent>
            {activeTab === 'overview' && (
              <OverviewTab indicator={indicator} />
            )}
            {activeTab === 'values' && (
              <ValuesTab
                values={values}
                loading={loading}
                onAddValue={handleAddValue}
              />
            )}
            {activeTab === 'disaggregation' && (
              <DisaggregationTab
                indicator={indicator}
                values={values}
              />
            )}
            {activeTab === 'history' && (
              <HistoryTab indicator={indicator} />
            )}
          </TabContent>
        </div>
      );
    };
    ```

  - Tips: Implement proper error handling and loading states
  - Acceptance Criteria:
    1. Components render correctly
    2. Form validation works properly
    3. API calls are handled correctly
    4. Error states are properly managed
    5. Loading states are properly shown
    6. User interactions are smooth
    7. Components are properly tested

## Additional Requirements

1. **Performance Optimization**
   - Implement caching for frequently accessed data
   - Optimize database queries
   - Add proper indexing
   - Acceptance Criteria:
     1. Response times are within acceptable limits
     2. Database queries are optimized
     3. Caching works correctly
     4. Performance metrics are monitored

2. **Security**
   - Implement proper authentication
   - Add rate limiting
   - Add input validation
   - Add proper error handling
   - Acceptance Criteria:
     1. Authentication works correctly
     2. Rate limiting is effective
     3. Input validation is thorough
     4. Error handling is secure

3. **Testing**
   - Add unit tests
   - Add integration tests
   - Add performance tests
   - Add security tests
   - Acceptance Criteria:
     1. Test coverage is above 80%
     2. All tests pass
     3. Performance tests meet requirements
     4. Security tests pass

4. **Documentation**
   - Add API documentation
   - Add code documentation
   - Add user documentation
   - Add deployment documentation
   - Acceptance Criteria:
     1. Documentation is complete
     2. Documentation is up-to-date
     3. Documentation is clear and helpful
     4. Documentation is properly formatted

## Acceptance Criteria

### 1. Indicator Creation and Configuration
- [ ] Users can create both quantitative and qualitative indicators
- [ ] All required fields are properly validated
- [ ] Indicator types are correctly enforced
- [ ] Calculation methods are properly configured
- [ ] Baseline and target values are correctly set
- [ ] Reporting frequencies are properly configured

### 2. Data Management
- [ ] Values can be entered manually and automatically
- [ ] Disaggregation works correctly for all indicator types
- [ ] Historical data is properly maintained
- [ ] Data validation prevents invalid entries
- [ ] Bulk imports work correctly
- [ ] Data linking with other modules functions properly

### 3. Calculation and Analysis
- [ ] Quantitative calculations are accurate
- [ ] Qualitative assessments are properly recorded
- [ ] Progress tracking works correctly
- [ ] Trend analysis provides accurate results
- [ ] Disaggregated analysis works as expected
- [ ] Custom formulas execute correctly

### 4. Integration
- [ ] Indicators link correctly with Plans module
- [ ] Activities module integration works properly
- [ ] Dashboard updates are real-time
- [ ] Reports include correct indicator data
- [ ] Export functionality works correctly
- [ ] API endpoints function as expected

### 5. Performance
- [ ] Indicator loading time is under 2 seconds
- [ ] Calculations are efficient
- [ ] Large datasets (>1000 items) are handled properly
- [ ] Caching reduces database load
- [ ] Real-time updates are responsive
- [ ] Export operations complete within acceptable time

### 6. Security
- [ ] Only authorized users can access indicators
- [ ] Data is properly validated
- [ ] Sensitive information is protected
- [ ] Audit logs are maintained
- [ ] Rate limiting is effective
- [ ] Input validation prevents attacks

### 7. User Experience
- [ ] Interface is intuitive and user-friendly
- [ ] Forms are easy to complete
- [ ] Error messages are clear and helpful
- [ ] Loading states are properly shown
- [ ] Mobile responsiveness works correctly
- [ ] Navigation is logical and efficient

### 8. Documentation
- [ ] API documentation is complete and accurate
- [ ] Code documentation follows standards
- [ ] User guides are clear and helpful
- [ ] Deployment documentation is thorough
- [ ] Examples are provided for common use cases
- [ ] Troubleshooting guides are available 