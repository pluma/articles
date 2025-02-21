# Implementing Union Types in Laravel Data

Ever wondered how you can use union types in Laravel Data? [According to the maintainers](https://github.com/spatie/laravel-data/discussions/743) you can't – at least for now. But after digging through the implementation details, I've found a workaround. Note that this will only work for unions of DTOs as unions with scalar types will behave differently.

Spatie's [Laravel Data package](https://spatie.be/docs/laravel-data/) is a powerful and popular choice for defining Data Transfer Objects (DTOs) in Laravel because it provides a rich feature set out of the box and makes it easy to define reusable data structures for situations where an Eloquent model is not sufficient or inappropriate.

In order to define a DTO with Laravel Data, you can create an ordinary class that extends `Spatie\LaravelData\Data` and specify all the attributes you want to be included as parameters to the constructor.

```php
class PerformerData extends \Spatie\LaravelData\Data
{
    public function __construct(
        public string $name,
        #[\Spatie\LaravelData\Attributes\Validation\In('Performer')]
        public string $objectType,
        /** @var ?list<string> */
        public ?array $instruments,
    ) {}
}

class RelatedPerformerData extends \Spatie\LaravelData\Data
{
    public function __construct(
        public PerformerData $performer,
        public string $relationship,
    ) {}
}
```

You can then use Laravel Data's attributes to specify additional validation rules for each parameter or to cast and transform the data going in or coming out.

But what if you want to use a union type for a parameter? Let's say you want to expand the `RelatedPerformerData` type to also allow representing bands consisting of multiple performers:

```php
class BandData extends \Spatie\LaravelData\Data
{
    public function __construct(
        public string $name,
        #[\Spatie\LaravelData\Attributes\Validation\In('Band')]
        public string $objectType,
        /** @var list<PerformerData> */
        public array $members,
    ) {}
}

class RelatedPerformerData extends \Spatie\LaravelData\Data
{
    public function __construct(
        public PerformerData|BandData $performer,
        public string $relationship,
    ) {}
}
```

At first glance, this will appear to work. When deserializing data from JSON or from an array with `performer` set to data that matches the `PerformerData` type, everything will still work as expected. But if we pass in data that matches the `BandData` type, the result will still be an `PerformerData` object:

```php
$relatedPerformer = RelatedPerformerData::from([
    'performer' => [
        'name' => 'The White Stripes',
        'objectType' => 'Band',
        'members' => [
            [
                'name' => 'Jack White',
                'objectType' => 'Performer',
                'instruments' => ['guitar', 'keyboards', 'piano', 'vocals'],
            ],
            [
                'name' => 'Meg White',
                'objectType' => 'Performer',
                'instruments' => ['drums', 'percussion', 'vocals'],
            ],
        ],
    ],
    'relationship' => 'performed_song',
]);

// RelatedPerformerData {
//     performer: PerformerData {
//         name: 'The White Stripes',
//         objectType: 'Band',
//         instruments: null,
//     },
//     relationship: 'performed_song',
// }
```

This will not cause an error in the current version of the package, though it will result in a validation error when using validation instead.

Laravel Data always uses the first type in a union of DTOs apparently. It records the union type as a `UnionType` internally because it has to parse composite types to handle the special `Optional` type for properties that can be omitted rather than nulled. But for the time being, it does not do anything with this information and always short-circuits.

Converting input data for a DTO seems like a good use case for using a `Cast`. We can create a custom cast that resolves the union type into one of the composite types. We could try to infer the correct type from the structure of the input data using validation, but that would be slow and error-prone given that two different types might accept the same input data, especially if all the distinguishing properties are optional.

Luckily our example scenario already has a property that uniquely identifies each of the two types: `objectType`. We can now use three strategies to decide how the object type of a given DTO should be determined:

1. The `objectType` property of the value DTO, expecting that it always uses a `Validation\In` attribute.
2. A custom static method on the value DTO returning the object type.
3. The class name of the value DTO, assuming it follows a naming convention.

In real-world use you should probably decide on a single strategy but for demonstration purposes we'll implement all three. We'll also want to decide how to handle the case where the `objectType` is not present in the input data and when none of the types in the union type qualifies.

Given that we're already using a `Validation\In` attribute, we can use the first strategy for the `objectType` property. The `Cast` interface provides a method signature that provides four inputs:

```php
public function cast(DataProperty $property, mixed $value, array $properties, CreationContext $context): mixed;
```

We can fetch the union's composite `$types` recursively from the `DataProperty` parameter with `array_keys($property->type->getAcceptedTypes())`. While Laravel Data also recognizes intersection types, we'll assume all combination types are union types for our purposes. We'll additionally assume that the `$value` will always be an array.

Getting the actual values passed to the validation attribute is a bit more challenging as that property is actually scoped as `private`. We can work around this by using reflection to get the value of the property. We'll use `DataConfig` to get the `DataClass` for each of the types. Laravel Data already uses reflection to generate each `DataClass` so this saves us from having to do it ourselves. From here we can get to the DTO's `objectType` property and its `Validation\In` attribute, which we then can use reflection with to get the values passed to the attribute:

```php
$objectType = $value['objectType'] ?? null;
$types = array_keys($propertyType->getAcceptedTypes());
$dataConfig = app(\Spatie\LaravelData\Support\DataConfig::class);
$reflectionClass = new \ReflectionClass(\Spatie\LaravelData\Attributes\Validation\In::class);
$valuesProperty = $reflectionClass->getProperty('values');
foreach ($types as $className) {
    $dataClass = $dataConfig->getDataClass($className);
    $objectTypeProperty = $dataClass->properties->get('objectType');
    $attribute = $objectTypeProperty?->attributes->first(
        fn (object $attribute) => $attribute instanceof \Spatie\LaravelData\Attributes\Validation\In
    );
    if (! $attribute) {
        continue;
    }
    $allowed = $valuesProperty->getValue($attribute);
    if (in_array($objectType, $allowed)) {
        return $className::from($value);
    }
}
```

For the second strategy, we'll need to add a static method to each of the DTOs. We can define an interface to to explicitly mark DTO classes that support this strategy:

```php
interface HasObjectType
{
    public static function getObjectType(): string;
}
```

This makes the second strategy fairly straightforward to implement:

```php
$objectType = $value['objectType'] ?? null;
$types = array_keys($propertyType->getAcceptedTypes());
foreach ($types as $className) {
    if (! is_subclass_of($className, HasObjectType::class)) {
        continue;
    }
    if ($objectType === $className::getObjectType()) {
        return $className::from($value);
    }
}
```

For the third strategy, we need to make sure that the DTO class name follows a convention that allows us to derive the object type from the class name. For our example, we'll simply assume that the object type always matches the class name, excluding the `Data` suffix if present:

```php
$objectType = $value['objectType'] ?? null;
$types = array_keys($propertyType->getAcceptedTypes());
foreach ($types as $className) {
    $localClassName = Str::of($className)->explode('\\')->last();
    $type = Str::replaceEnd('Data', '', $localClassName);
    if ($objectType === $type) {
        return $className::from($value);
    }
}
```

Let's put this all together and extract the logic into a reusable cast. We'll default to returning the `dataClass` of the property type, which will be the first DTO type in the union, to match the default behavior of Laravel Data.

We'll also make the property name configurable in case we later want to use a different property name for the object type. Finally, we'll also extract the resolution logic into a public method for reasons you'll find out very soon.

```php
use Illuminate\Support\Str;
use Spatie\LaravelData\Attributes\Validation;
use Spatie\LaravelData\Casts\Cast;
use Spatie\LaravelData\Support\Creation\CreationContext;
use Spatie\LaravelData\Support\DataConfig;
use Spatie\LaravelData\Support\DataProperty;
use Spatie\LaravelData\Support\DataPropertyType;

class ObjectTypeCast implements Cast
{
    public function __construct(
        protected string $objectTypeProperty = 'objectType',
    ) {}

    public function resolveObjectType(mixed $value, DataPropertyType $propertyType): string
    {
        $objectType = $value[$this->objectTypeProperty] ?? null;
        $defaultType = $propertyType->dataClass;

        if (! isset($objectType)) {
            return $defaultType;
        }

        $dataConfig = app(DataConfig::class);
        $reflectionClass = new \ReflectionClass(Validation\In::class);
        $valuesProperty = $reflectionClass->getProperty('values');

        $types = array_keys($propertyType->getAcceptedTypes());
        foreach ($types as $className) {
            $dataClass = $dataConfig->getDataClass($className);
            $objectTypeProperty = $dataClass->properties->get($this->objectTypeProperty);
            $attribute = $objectTypeProperty?->attributes->first(
                fn (object $attribute) => $attribute instanceof Validation\In
            );
            if ($attribute) {
                $allowed = $valuesProperty->getValue($attribute);
                if (in_array($objectType, $allowed)) {
                    return $className;
                }
            } elseif (is_subclass_of($className, HasObjectType::class)) {
                $type = $className::getObjectType();
                if ($type === $objectType) {
                    return $className;
                }
            } else {
                $localClassName = Str::of($className)->explode('\\')->last();
                $type = Str::replaceEnd('Data', '', $localClassName);
                if ($objectType === $type) {
                    return $className;
                }
            }
        }

        return $defaultType;
    }

    public function cast(DataProperty $property, mixed $value, array $properties, CreationContext $context): mixed
    {
        $type = $this->resolveObjectType($value, $property->type);
        return $type::from($value);
    }
}
```

We can now use this new cast to augment our `RelatedPerformerData` DTO:

```php
use Spatie\LaravelData\Attributes\WithCast;

class RelatedPerformerData extends \Spatie\LaravelData\Data
{
    public function __construct(
        #[WithCast(ObjectTypeCast::class, 'objectType')]
        public PerformerData|BandData $performer,
        public string $relationship,
    ) {}
}
```

This will now work as expected, however we will still see the same issue as before when using validation.

Laravel Data actually ignores the casts when inferring validation rules. While Laravel Data provides ways to register additional rule inferrers, those do not let us modify the handling of properties that use DTOs or DTO union types.

As with getting the value of the `private` property of the `Validation\In` attribute, we can use some tricks to work around this. In this case, we'll replace Laravel Data's default `DataValidationRulesResolver` with our own implementation that knows about `ObjectTypeCast`.

Let's first create a new class that extends Laravel Data's `DataValidationRulesResolver` and override the `resolveDataObjectSpecificRules` method:

```php
use Illuminate\Support\Arr;
use Spatie\LaravelData\Resolvers\DataValidationRulesResolver;
use Spatie\LaravelData\Support\DataProperty;
use Spatie\LaravelData\Support\Validation\DataRules;
use Spatie\LaravelData\Support\Validation\ValidationPath;

class ObjectTypeAwareDataValidationRulesResolver extends DataValidationRulesResolver
{
    protected function resolveDataObjectSpecificRules(
        DataProperty $dataProperty,
        array $fullPayload,
        ValidationPath $path,
        ValidationPath $propertyPath,
        DataRules $dataRules,
    ): void {
        $this->resolveToplevelRules(
            $dataProperty,
            $fullPayload,
            $path,
            $propertyPath,
            $dataRules
        );
        $dataClass = $dataProperty->type->dataClass;
        if ($dataProperty->cast instanceof ObjectTypeCast) {
            $value = Arr::get($fullPayload, $propertyPath->get());
            $dataClass = $dataProperty->cast->resolveObjectType($value, $dataProperty->type);
        }
        $this->execute(
            $dataClass,
            $fullPayload,
            $propertyPath,
            $dataRules,
        );
    }
}
```

Normally the method would pass `$dataProperty->type->dataClass` directly to `$this->execute`, which matches the default behavior we also made sure to fall back to in our cast. But we've added a check for whether the property has an `ObjectTypeCast` and if so, we'll let it resolve the correct type instead.

Note that we need to fetch the actual value from the payload as we're only interested in the value of the property itself, not the entire payload Laravel Data is trying to process.

We can now replace Laravel Data's default `DataValidationRulesResolver` with our own implementation by substituting it in our service provider's `boot` method:

```php
app()->singleton(
    \Spatie\LaravelData\Resolvers\DataValidationRulesResolver::class,
    ObjectTypeAwareDataValidationRulesResolver::class,
);
```

Now both the input casting and the validation will resolve union types correctly as long as they are using the `ObjectTypeCast`.

As always, it is important to remember the limitations of this implementation and the assumptions we made in the beginning. Eventually Laravel Data may have built-in support for union types making this workaround unnecessary. Until then, this may help you in real-world scenarios where they may be unavoidable.

(CC BY-NC-ND 4.0) 2025 Alan Plum
