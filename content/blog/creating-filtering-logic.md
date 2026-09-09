---
title: From If-Chains to Design Patterns
description: Notes on how a property-search filter engine evolved, from a naive if-chain through Strategy, Composite, and classic design patterns.
date: 2026-09-07
tags: java design-patterns
---

Notes on how a property\-search filter engine evolved, kept mostly for future\-me. Each step is the direct consequence of a problem the previous step created — not a pattern applied for its own sake.

## 1\. The naive version: one class, if\-chains

- Everything lived on `Property` as raw fields, and search was a single method with a growing pile of conditionals.
- No `Filter` abstraction, no `Field` identity, no criterion object — just a method that knows every filterable attribute by name and checks them all inline.

```java
class Property {
    String id;
    double price;
    double areaSqft;
    String bhk;
    String propertyType;    // "APARTMENT", "VILLA"
    String furnishing;      // "FURNISHED", "SEMI_FURNISHED", "UNFURNISHED"
    Set<String> amenities;  // {"GYM", "POOL", "PARKING"}
    GeoPoint location;
    String postedBy;        // "OWNER", "AGENT"
}

class BookingPlatform {
    List<Property> properties = new ArrayList<>();

    List<Property> search(double minPrice, double maxPrice, String bhk, String amenity) {
        List<Property> results = new ArrayList<>();
        for (Property p : properties) {
            if (p.price < minPrice || p.price > maxPrice) continue;
            if (!p.bhk.equals(bhk)) continue;
            if (!p.amenities.contains(amenity)) continue;
            results.add(p);
        }
        return results;
    }
}
```

- This works fine for exactly the query shapes the method author anticipated, and falls apart the moment a caller wants anything else.

- **The signature grows without bound.** Every new filterable field (furnishing, property type, posted\-by, geo radius...) is another parameter:
  
  ```java
  List<Property> search(double minPrice, double maxPrice, String bhk,
                         String furnishing, String propertyType,
                         String amenity, GeoPoint center, double radiusKm,
                         String postedBy) { ... }
  ```
  
  A method with nine\-plus parameters is unreadable, and every call site becomes a wall of positional arguments — easy to pass `bhk` where `furnishing` belongs and not notice, since both are `String`.

- **Optional filters need sentinel values or overloads.** If a caller doesn't want to filter by `bhk`, what do they pass — `null`? An empty string? A separate overloaded method per combination of "which filters are active"? Each option is its own kind of footgun, and the overload count explodes combinatorially.

- **Combinators are impossible.** "Price OR bhk" instead of "price AND bhk" isn't expressible — the method hardcodes AND semantics between every `if` check. Getting OR support means writing an entirely parallel method, and nested AND/OR groups mean writing bespoke code per query shape, forever.

- **Nothing is reusable or testable in isolation.** The range\-check logic for price (`p.price >= minPrice && p.price <= maxPrice`) can't be unit\-tested on its own — it only exists inline, inside one big method, coupled to iterating the whole property list.

- **Data and behavior are mixed.** `Property` is just a bag of fields; the only place that knows what a valid query looks like is the search method itself. There's no object that represents "a single filter condition" that could be constructed, passed around, or logged independently.

Every step after this one exists to remove one of these constraints without reintroducing the others.

## 2\. Filters as Strategy objects — but field access still hardcoded

- Pulled each condition into its own object implementing a shared `Filter` interface, so filters could be constructed independently, held in a list, and swapped in without touching a search method's signature at all.
- This is the Strategy pattern: a single abstraction (`Filter`), many interchangeable implementations, and calling code that only ever depends on the abstraction.

```java
interface Filter {
    boolean isSatisfiedBy(Property property);
}

class PriceRangeFilter implements Filter {
    private final double min;
    private final double max;

    PriceRangeFilter(double min, double max) {
        this.min = min;
        this.max = max;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        return p.price >= min && p.price <= max;   // hardcoded field access
    }
}

class AreaRangeFilter implements Filter {
    private final double min;
    private final double max;

    AreaRangeFilter(double min, double max) {
        this.min = min;
        this.max = max;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        return p.areaSqft >= min && p.areaSqft <= max;   // identical logic, different field
    }
}

class GeoRadiusFilter implements Filter {
    private final GeoPoint center;
    private final double radiusKm;

    GeoRadiusFilter(GeoPoint center, double radiusKm) {
        this.center = center;
        this.radiusKm = radiusKm;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        return distanceKm(center, p.location) <= radiusKm;   // hardcoded call to p.location
    }
}
```

- `BookingPlatform.search` can now take a `List<Filter>` and AND them together generically:
  
  ```java
  List<Property> search(List<Filter> filters) {
      return properties.stream()
          .filter(p -> filters.stream().allMatch(f -> f.isSatisfiedBy(p)))
          .collect(toList());
  }
  ```

- This genuinely fixes the combinator problem — a `List<Filter>` can be ANDed (or, with a small change, ORed) by whatever code holds it, without any `Filter` implementation knowing about any other.

- But look at `PriceRangeFilter` and `AreaRangeFilter` side by side: **identical logic, duplicated**, differing only in which field they read (`p.price` vs `p.areaSqft`). Every new range\-filterable field means copy\-pasting a whole class.

- `GeoRadiusFilter.isSatisfiedBy` calls `p.location` directly, baked into the class — **the filter can't be reused for a field it wasn't written for.** If `Property` later had a second `GeoPoint` field (say, a "nearest metro station" location), a whole new filter class would be needed even though the radius\-check math is identical.

- This exact problem was called out directly as a fix in the real codebase's history: *"`GeoRadiusFilter` now goes through the same extractor pattern as every other filter type, instead of hardcoding `property.getLocation()` internally."* That's exactly what the next step solves.

## 3\. `Field` \+ `FieldExtractor` — decoupling "which field" from "how to read it"

- Introduced a `Field` enum naming every filterable attribute, and a one\-method functional interface `FieldExtractor<T>` that knows how to pull one typed value out of a `Property`.
- An `Extractors` factory maps `Field → FieldExtractor`, split into four typed methods by the shape of value they return.

```java
enum Field { PRICE, AREA, BHK, PROPERTY_TYPE, FURNISHING, AMENITIES, LOCATION, POSTED_BY }

interface FieldExtractor<T> {
    T extract(Property property);
}

final class Extractors {
    private Extractors() {}

    static FieldExtractor<Double> numeric(Field field) {
        return switch (field) {
            case PRICE -> Property::getPrice;
            case AREA  -> Property::getAreaSqft;
            default -> throw new IllegalArgumentException("Not a numeric field: " + field);
        };
    }

    static FieldExtractor<String> categorical(Field field) {
        return switch (field) {
            case BHK           -> Property::getBhk;
            case FURNISHING    -> Property::getFurnishing;
            case PROPERTY_TYPE -> Property::getPropertyType;
            case POSTED_BY     -> Property::getPostedBy;
            default -> throw new IllegalArgumentException("Not a categorical field: " + field);
        };
    }

    static FieldExtractor<Set<String>> multiValued(Field field) {
        return switch (field) {
            case AMENITIES -> Property::getAmenities;
            default -> throw new IllegalArgumentException("Not a multi-valued field: " + field);
        };
    }

    static FieldExtractor<GeoPoint> geo(Field field) {
        return switch (field) {
            case LOCATION -> Property::getLocation;
            default -> throw new IllegalArgumentException("Not a geo field: " + field);
        };
    }
}
```

- Now there's exactly one `Filter` implementation per *value shape*, not per field — the field it reads is an injected constructor argument:

```java
class RangeFilter implements Filter {
    private final double min, max;
    private final FieldExtractor<Double> extractor;

    RangeFilter(double min, double max, FieldExtractor<Double> extractor) {
        this.min = min; this.max = max; this.extractor = extractor;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        double v = extractor.extract(p);
        return v >= min && v <= max;
    }
}

class MultiSelectFilter implements Filter {
    private final Set<String> allowedValues;
    private final FieldExtractor<String> extractor;

    MultiSelectFilter(Set<String> allowedValues, FieldExtractor<String> extractor) {
        this.allowedValues = allowedValues; this.extractor = extractor;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        return allowedValues.contains(extractor.extract(p));
    }
}

class GeoRadiusFilter implements Filter {
    private final GeoPoint center;
    private final double radiusKm;
    private final DistanceCalculator distanceCalculator;
    private final FieldExtractor<GeoPoint> extractor;

    GeoRadiusFilter(GeoPoint center, double radiusKm, DistanceCalculator distanceCalculator,
                     FieldExtractor<GeoPoint> extractor) {
        this.center = center; this.radiusKm = radiusKm;
        this.distanceCalculator = distanceCalculator; this.extractor = extractor;
    }

    @Override
    public boolean isSatisfiedBy(Property p) {
        return distanceCalculator.distanceKm(center, extractor.extract(p)) <= radiusKm;
    }
}
```

- One `RangeFilter` now handles both price and area — construct it with `Extractors.numeric(Field.PRICE)` or `Extractors.numeric(Field.AREA)`, no new class either way.
- Each `Extractors` method is a `switch` on `Field` with a `default -> throw new IllegalArgumentException(...)` — so asking for the wrong\-shaped extractor for a field (e.g. the numeric extractor for `BHK`) fails loudly, immediately, rather than compiling into nonsense.
- What this *doesn't* fix: `RangeFilter`'s constructor takes any `FieldExtractor<Double>` — nothing stops constructing `new RangeFilter(min, max, Extractors.numeric(Field.AREA))` when the caller meant to build a price filter. `Extractors` only guards against the wrong\-*shaped* extractor for a field, not a valid\-but\-semantically\-mismatched one. That gap narrows in step 5, once criteria and filters are wired together through a single factory instead of by hand.
- If `Field`'s identity here had stayed a raw `String` (`"price"`, `"PRICE"`, `"Price"`...) instead of an enum, a typo becomes a runtime `IllegalArgumentException` discovered only when that exact query path runs — not a compile error. The enum turns that failure mode off entirely; this itself was a fix over an earlier raw\-string version of `Field`.

## 4\. Criterion as one class with nullable fields — then the NPE trap

- The client\-facing "what did the caller ask for" object (`FilterCriterion`) started as a single class holding every field any filter type might need — all nullable, only the relevant ones populated per filter type.

```java
class FilterCriterion {
    Field field;
    FilterType type;
    Double min, max;        // set only for RANGE
    Set<String> values;     // set only for MULTI_SELECT
    Set<String> amenities;  // set only for AMENITIES
    GeoPoint center;         // set only for GEO_RADIUS
    double radiusKm;         // set only for GEO_RADIUS
}
```

- Calling `getMin()` on a criterion actually built as `MULTI_SELECT` doesn't fail to compile — it just returns `null`, and blows up later, wherever that `null` gets used arithmetically (`NullPointerException`, possibly far from where the mistake was made).
- The class can't enforce "these fields only make sense together" — Java has no way to express that constraint within one flat class with optional fields.
- **The fix:** split into one interface plus four concrete classes, each holding only the fields it actually needs, each validating its own invariants at construction.

```java
enum FilterType { RANGE, MULTI_SELECT, AMENITIES, GEO_RADIUS }

interface FilterCriterion {
    FilterType getType();
    Field getField();

    static FilterCriterion range(Field field, double min, double max) {
        return new RangeCriterion(field, min, max);
    }
    static FilterCriterion multiSelect(Field field, Set<String> values) {
        return new MultiSelectCriterion(field, values);
    }
    static FilterCriterion amenities(Set<String> values) {
        return new AmenitiesCriterion(values);
    }
    static FilterCriterion geoRadius(GeoPoint center, double radiusKm) {
        return new GeoRadiusCriterion(center, radiusKm);
    }
}

class RangeCriterion implements FilterCriterion {
    private final Field field;
    private final double min, max;

    RangeCriterion(Field field, double min, double max) {
        if (min > max) throw new IllegalArgumentException("min must be <= max");
        this.field = field; this.min = min; this.max = max;
    }

    @Override public FilterType getType() { return FilterType.RANGE; }
    @Override public Field getField() { return field; }
    public double getMin() { return min; }
    public double getMax() { return max; }
}

class MultiSelectCriterion implements FilterCriterion {
    private final Field field;
    private final Set<String> values;

    MultiSelectCriterion(Field field, Set<String> values) {
        if (values == null || values.isEmpty()) throw new IllegalArgumentException("values required");
        this.field = field; this.values = Set.copyOf(values);
    }

    @Override public FilterType getType() { return FilterType.MULTI_SELECT; }
    @Override public Field getField() { return field; }
    public Set<String> getValues() { return values; }
}

class AmenitiesCriterion implements FilterCriterion {
    private final Set<String> requiredAmenities;

    AmenitiesCriterion(Set<String> requiredAmenities) {
        if (requiredAmenities == null || requiredAmenities.isEmpty())
            throw new IllegalArgumentException("requiredAmenities required");
        this.requiredAmenities = Set.copyOf(requiredAmenities);
    }

    @Override public FilterType getType() { return FilterType.AMENITIES; }
    @Override public Field getField() { return Field.AMENITIES; }   // always this field — no ambiguity to resolve
    public Set<String> getRequiredAmenities() { return requiredAmenities; }
}

class GeoRadiusCriterion implements FilterCriterion {
    private final GeoPoint center;
    private final double radiusKm;

    GeoRadiusCriterion(GeoPoint center, double radiusKm) {
        if (center == null) throw new IllegalArgumentException("center required");
        if (radiusKm <= 0) throw new IllegalArgumentException("radiusKm must be > 0");
        this.center = center; this.radiusKm = radiusKm;
    }

    @Override public FilterType getType() { return FilterType.GEO_RADIUS; }
    @Override public Field getField() { return Field.LOCATION; }
    public GeoPoint getCenter() { return center; }
    public double getRadiusKm() { return radiusKm; }
}
```

- `AmenitiesCriterion`/`GeoRadiusCriterion` hardcode `getField()` rather than taking a `Field` parameter, because there's exactly one field of that shape in the whole model — no second geo field or amenities field to disambiguate. `RangeCriterion`/`MultiSelectCriterion` do take `Field`, because a range or multi\-select check could legitimately apply to more than one field (price *or* area; bhk *or* furnishing *or* property type).
- Each concrete constructor is **package\-private**, not public — `new RangeCriterion(...)` is never called from outside the package. The static factory methods on `FilterCriterion` are the only public entry point.
- This buys a specific, load\-bearing guarantee: `getType()` returning `RANGE` and the object's actual runtime class being `RangeCriterion` are set together, atomically, in the same constructor call — there is no code path that could produce one without the other. That guarantee is what the next step's unchecked cast relies on.
- At this point it was a **convention\-enforced** guarantee, not a compiler\-enforced one: nothing stopped another class in the same package from implementing `FilterCriterion` directly and returning a mismatched `getType()`. Step 10 closes exactly this gap by sealing the interface.

## 5\. `FilterFactory` — bridging descriptive criteria to executable filters

- `FilterCriterion` (data — what the caller asked for) and `Filter` (behavior — something that can evaluate itself against a `Property`) are deliberately two separate hierarchies.
- Something has to convert one into the other. That conversion is the entire job of `FilterFactory` / `DefaultFilterFactory` — nothing more, nothing less.

```java
interface FilterFactory {
    Filter create(FilterCriterion criterion);
}

class DefaultFilterFactory implements FilterFactory {
    private final DistanceCalculator distanceCalculator;

    DefaultFilterFactory(DistanceCalculator distanceCalculator) {
        this.distanceCalculator = distanceCalculator;
    }

    @Override
    public Filter create(FilterCriterion criterion) {
        return switch (criterion.getType()) {
            case RANGE -> {
                RangeCriterion c = (RangeCriterion) criterion;
                yield new RangeFilter(c.getMin(), c.getMax(), Extractors.numeric(c.getField()));
            }
            case MULTI_SELECT -> {
                MultiSelectCriterion c = (MultiSelectCriterion) criterion;
                yield new MultiSelectFilter(c.getValues(), Extractors.categorical(c.getField()));
            }
            case AMENITIES -> {
                AmenitiesCriterion c = (AmenitiesCriterion) criterion;
                yield new AmenitiesFilter(c.getRequiredAmenities(), Extractors.multiValued(c.getField()));
            }
            case GEO_RADIUS -> {
                GeoRadiusCriterion c = (GeoRadiusCriterion) criterion;
                yield new GeoRadiusFilter(c.getCenter(), c.getRadiusKm(), distanceCalculator,
                                            Extractors.geo(c.getField()));
            }
        };
    }
}
```

- Each branch does the same three things: cast the interface reference down to the concrete criterion type, read the plain data off it, and use that data — plus a `FieldExtractor` pulled from `Extractors` — to construct the matching `Filter`.
- This is also where step 3's mismatch gap closes: the factory always calls `Extractors.numeric(c.getField())` using the *same* `Field` the criterion itself carries — a `RangeCriterion` built for `PRICE` can never end up wired to an `AREA` extractor by accident, because the caller never gets to choose the extractor independently of the field.
- The cast in each branch (`(RangeCriterion) criterion`) looks like a classic runtime\-risk smell — four opportunities for `ClassCastException` if a `FilterType` case ever disagreed with the object's real class.
- It doesn't, precisely because of step 4's construction guarantee: there's no way to produce a `FilterCriterion` whose `getType()` and concrete class disagree, so the cast is safe *by construction*, not by luck.
- Only one `FilterFactory` implementation exists. That's arguably speculative abstraction — an interface with a single implementation and no test double in sight is often a sign the interface wasn't strictly necessary. Kept anyway here because: (a) it lets code that depends on `FilterFactory` be tested against a stub without touching real extractor wiring; (b) a `CachingFilterFactory` or `ValidatingFilterFactory` wrapping the default one isn't a stretch to imagine; (c) it matches the same interface\-plus\-default\-impl shape used everywhere else in this codebase (`Filter`, `DistanceCalculator`).

## 6\. Composite \+ Builder — nesting AND/OR, but the request stayed flat

- `CompositeFilter` implements `Filter` itself, and holds a list of child `Filter`s combined with an operator — this is the Composite pattern: a container of `Filter`s that is *also* a `Filter`.

```java
enum LogicalOperator { AND, OR }

class CompositeFilter implements Filter {
    private final LogicalOperator operator;
    private final List<Filter> children = new ArrayList<>();

    CompositeFilter(LogicalOperator operator) {
        this.operator = operator;
    }

    CompositeFilter add(Filter filter) {
        children.add(filter);
        return this;
    }

    @Override
    public boolean isSatisfiedBy(Property property) {
        return switch (operator) {
            case AND -> children.stream().allMatch(f -> f.isSatisfiedBy(property));
            case OR  -> children.stream().anyMatch(f -> f.isSatisfiedBy(property));
        };
    }
}
```

- Because a `CompositeFilter` *is* a `Filter`, composites nest inside composites — an OR of two ANDs, an AND containing an OR containing more ANDs — and the recursion just works, since every level only ever calls `isSatisfiedBy` on its children without caring whether a child is a leaf filter or another composite.
- `FilterRequestBuilder` assembles these trees from a flat list of criteria, running each one through a `FilterFactory` and adding the result to a root `CompositeFilter`\:

```java
class FilterRequestBuilder {
    private final FilterFactory filterFactory;
    private final CompositeFilter root;

    FilterRequestBuilder(FilterFactory filterFactory) {
        this(filterFactory, LogicalOperator.AND);   // default root: AND
    }

    FilterRequestBuilder(FilterFactory filterFactory, LogicalOperator rootOperator) {
        this.filterFactory = filterFactory;
        this.root = new CompositeFilter(rootOperator);
    }

    FilterRequestBuilder withCriterion(FilterCriterion criterion) {
        root.add(filterFactory.create(criterion));
        return this;
    }

    FilterRequestBuilder withCriteria(List<FilterCriterion> criteria) {
        criteria.forEach(this::withCriterion);
        return this;
    }

    FilterRequestBuilder withGroup(Filter group) {   // attach a pre-built sub-tree
        root.add(group);
        return this;
    }

    Filter build() {
        return root;
    }

    static Filter group(FilterFactory filterFactory, LogicalOperator operator, List<FilterCriterion> criteria) {
        CompositeFilter group = new CompositeFilter(operator);
        criteria.forEach(c -> group.add(filterFactory.create(c)));
        return group;
    }
}
```

- Building a nested `(bhk=1 AND price<25L) OR (bhk=4+ AND hasGarden)` query, at this point in the evolution, looked like this — note it never touches `SearchRequest` at all:

```java
Filter cheap = FilterRequestBuilder.group(factory, LogicalOperator.AND, List.of(
        FilterCriterion.multiSelect(Field.BHK, Set.of("1")),
        FilterCriterion.range(Field.PRICE, 0, 2_500_000)));
Filter luxury = FilterRequestBuilder.group(factory, LogicalOperator.AND, List.of(
        FilterCriterion.multiSelect(Field.BHK, Set.of("4+")),
        FilterCriterion.amenities(Set.of("GARDEN"))));

Filter orOfGroups = new FilterRequestBuilder(factory, LogicalOperator.OR)
        .withGroup(cheap)
        .withGroup(luxury)
        .build();

platform.search(orOfGroups);   // via the raw-Filter overload, not SearchRequest
```

- **Gap 1, fixed:** the root operator wasn't configurable at first — always `AND` — so there was no way to express a top\-level `OR`. Fixed by making it a constructor argument.
- **Gap 2, not yet fixed here:** `SearchRequest`, the actual client\-facing request object, stayed a flat `List<FilterCriterion>` even after gap 1's fix. Expressing the nested query above meant bypassing `SearchRequest` entirely and talking to `Filter`/`FilterRequestBuilder` directly — which defeats a good part of the point of having a request DTO. Step 9 is what actually closes this.

## 7\. Hardening `Property` — immutability, defensive copies, closed\-value enums, and safe concurrent reads

Several independent hardening passes, grouped here because they share one theme: making it structurally hard to end up with corrupted or inconsistent state.

**Immutability \+ defensive copying**

```java
class Property {
    private final String id;
    private final double price;
    private final double areaSqft;
    private final String bhk;
    private final PropertyType propertyType;
    private final Furnishing furnishing;
    private final Set<Amenity> amenities;
    private final GeoPoint location;
    private final PostedBy postedBy;

    Property(String id, double price, double areaSqft, String bhk, PropertyType propertyType,
             Furnishing furnishing, Set<Amenity> amenities, GeoPoint location, PostedBy postedBy) {
        this.id = id;
        this.price = price;
        this.areaSqft = areaSqft;
        this.bhk = bhk;
        this.propertyType = propertyType;
        this.furnishing = furnishing;
        this.amenities = Collections.unmodifiableSet(EnumSet.copyOf(amenities));   // <- defensive copy
        this.location = location;
        this.postedBy = postedBy;
    }
    // getters only — no setters, every field final
}
```

- Every field is `final`, set once, no setters — once a `Property` exists, nothing can mutate it later. This matters once it's sitting in a list read from concurrently (see below).
- The constructor does **not** store the `Set<Amenity>` it's handed as\-is. Without `Collections.unmodifiableSet(EnumSet.copyOf(...))`, a caller who kept a reference to the set they passed in could mutate it after construction and silently corrupt a `Property` that's supposed to be immutable — the bug shows up later, wherever the corrupted `Property` gets read, not where the mutation happened.
- `EnumSet` is also more compact than a general `HashSet<String>` now that `Amenity` is a closed enum — it's backed by a bitmask, not hash buckets.

**Raw strings to enums for closed categories**

- `propertyType`, `furnishing`, `postedBy`, and `amenities` all started as `String` / `Set<String>`. If they'd stayed strings, `"APARTMNT"` compiles fine and fails at query time, deep inside whatever code tries to match against it — not at the call site where the typo was actually made.
- Converted one at a time to `PropertyType`, `Furnishing`, `PostedBy`, and `Amenity` enums, re\-verifying the demo ran identically after each.
- `bhk` deliberately stayed a raw `String` — `"4+"` isn't a valid Java enum\-constant identifier, so representing it as an enum means an awkward constant like `FOUR_PLUS` plus a separate display\-string field. More ceremony than the payoff is worth for a value that's "a bounded string with one irregular case," not a clean closed category.
- The conversion had one ripple effect: `Extractors.categorical()` (step 3) returns `FieldExtractor<String>`, because that return type is shared, generic machinery across every categorical field — and `BHK` genuinely is, and stays, a string. So the enum\-backed fields bridge through `.name()` instead of returning their getter directly:

```java
static FieldExtractor<String> categorical(Field field) {
    return switch (field) {
        case BHK           -> Property::getBhk;               // still a raw string
        case FURNISHING    -> p -> p.getFurnishing().name();    // enum -> String bridge
        case PROPERTY_TYPE -> p -> p.getPropertyType().name();
        case POSTED_BY     -> p -> p.getPostedBy().name();
        default -> throw new IllegalArgumentException("Not a categorical field: " + field);
    };
}

static FieldExtractor<Set<Amenity>> multiValued(Field field) {
    return switch (field) {
        case AMENITIES -> Property::getAmenities;   // now Set<Amenity>, not Set<String>
        default -> throw new IllegalArgumentException("Not a multi-valued field: " + field);
    };
}
```

- `Property` itself stays fully typed; this `.name()` bridge, contained entirely inside `Extractors`, is the one seam where that typing gets deliberately flattened back to a string so the rest of the generic `MultiSelectFilter`/`MultiSelectCriterion` machinery didn't need rewriting.
- `AmenitiesFilter`, `AmenitiesCriterion`, and `FilterCriterion.amenities(...)` all changed from `Set<String>` to `Set<Amenity>` in lockstep with this — amenities checking is fully typed end\-to\-end now, unlike the bridged categorical fields, because that extractor is only ever used for one field (`AMENITIES`).

**Thread\-safe reads**

```java
class BookingPlatform {
    private final List<Property> properties = new CopyOnWriteArrayList<>();   // not ArrayList
    // ...
}
```

- Reads (`search()`) are far more frequent than writes (`addProperty`/`removeProperty`) in a system shaped like this — exactly the access pattern `CopyOnWriteArrayList` is built for.
- With a plain `ArrayList`, a write landing during an in\-flight `search()` iteration throws `ConcurrentModificationException`. `CopyOnWriteArrayList` makes that impossible by copying the whole backing array on every write and letting iterators keep working against their own snapshot — at the cost of every write being O(n).

## 9\. `SearchNode` — closing the last gap: nested trees as pure data

- `SearchRequest` held a flat `List<FilterCriterion>` right up through step 6's fix to the builder — meaning nested AND/OR was only reachable by dropping down to `Filter`/`FilterRequestBuilder` directly, which defeats the purpose of having a request DTO at all.
- Fixed by giving `SearchRequest` a `SearchNode` root instead of a list — a small Composite at the *descriptive* layer, mirroring `CompositeFilter`'s shape (step 6) one layer up, before anything is wired to a `FieldExtractor` or `FilterFactory`.

```java
sealed interface SearchNode permits CriterionNode, GroupNode {
    static SearchNode criterion(FilterCriterion criterion) {
        return new CriterionNode(criterion);
    }
    static SearchNode group(LogicalOperator operator, List<SearchNode> children) {
        return new GroupNode(operator, children);
    }
}

final class CriterionNode implements SearchNode {
    private final FilterCriterion criterion;

    CriterionNode(FilterCriterion criterion) {
        if (criterion == null) throw new IllegalArgumentException("criterion required");
        this.criterion = criterion;
    }

    FilterCriterion getCriterion() { return criterion; }
}

final class GroupNode implements SearchNode {
    private final LogicalOperator operator;
    private final List<SearchNode> children;

    GroupNode(LogicalOperator operator, List<SearchNode> children) {
        if (operator == null) throw new IllegalArgumentException("operator required");
        if (children == null || children.isEmpty()) throw new IllegalArgumentException("children required");
        this.operator = operator;
        this.children = List.copyOf(children);
    }

    LogicalOperator getOperator() { return operator; }
    List<SearchNode> getChildren() { return children; }
}
```

- A `SearchRequestCompiler` recursively turns a `SearchNode` tree into a `Filter` tree — one `CompositeFilter` per `GroupNode`, one `filterFactory.create(...)` call per `CriterionNode` leaf:

```java
final class SearchRequestCompiler {
    private SearchRequestCompiler() {}

    static Filter compile(SearchNode node, FilterFactory filterFactory) {
        return switch (node) {
            case CriterionNode c -> filterFactory.create(c.getCriterion());
            case GroupNode g -> {
                CompositeFilter composite = new CompositeFilter(g.getOperator());
                g.getChildren().forEach(child -> composite.add(compile(child, filterFactory)));
                yield composite;
            }
        };
    }
}
```

- Because `SearchNode` is declared `sealed permits CriterionNode, GroupNode`, this `switch` has **no `default` branch** and is exhaustive by construction — the compiler checks it, not a convention. Adding a third `SearchNode` subtype later without updating this method is a compile error, not a silent gap where the new type falls through unhandled. This is step 5's cast\-safety idea, upgraded from "safe by convention" to "safe because the compiler verified it."
- `SearchRequest` itself, plus two convenience factories for the common flat case:

```java
class SearchRequest {
    private final SearchNode root;

    SearchRequest(SearchNode root) { this.root = root; }

    SearchNode getRoot() { return root; }

    static SearchRequest allOf(FilterCriterion... criteria) {
        return new SearchRequest(SearchNode.group(LogicalOperator.AND, toNodes(criteria)));
    }
    static SearchRequest anyOf(FilterCriterion... criteria) {
        return new SearchRequest(SearchNode.group(LogicalOperator.OR, toNodes(criteria)));
    }
    private static List<SearchNode> toNodes(FilterCriterion... criteria) {
        return Arrays.stream(criteria).map(SearchNode::criterion).toList();
    }
}

class BookingPlatform {
    List<Property> search(SearchRequest request) {
        Filter filter = SearchRequestCompiler.compile(request.getRoot(), filterFactory);
        return filterEngine.apply(filter, properties);
    }
}
```

- The same OR\-of\-two\-ANDs query from step 6, now expressed as data through `SearchRequest` instead of by calling the builder directly:

```java
SearchNode cheap = SearchNode.group(LogicalOperator.AND, List.of(
        SearchNode.criterion(FilterCriterion.multiSelect(Field.BHK, Set.of("1"))),
        SearchNode.criterion(FilterCriterion.range(Field.PRICE, 0, 2_500_000))));
SearchNode luxury = SearchNode.group(LogicalOperator.AND, List.of(
        SearchNode.criterion(FilterCriterion.multiSelect(Field.BHK, Set.of("4+"))),
        SearchNode.criterion(FilterCriterion.amenities(Set.of(Amenity.GARDEN)))));

SearchRequest request = new SearchRequest(SearchNode.group(LogicalOperator.OR, List.of(cheap, luxury)));
platform.search(request);   // one overload, same as any flat query — no Filter or builder in sight
```

- `allOf(...)`/`anyOf(...)` keep the common flat case exactly as easy as it always was — they just wrap the criteria in an implicit single\-level `GroupNode`, so this change costs nothing for the 90% case and only asks for the extra `SearchNode.group(...)`/`SearchNode.criterion(...)` wrapping when a query actually needs nesting.
- If this were still a flat list, arbitrary nested boolean queries would remain something only internal code could build by hand. As sealed, recursive data, they're something a client — or a deserialized JSON request body, in a real system — could construct directly, and the raw `search(Filter)` overload that used to be the *only* way to run such a query becomes optional rather than load\-bearing.

## 10\. Negation, and sealing `FilterCriterion` for real

Two separate additions, both closing gaps flagged earlier rather than opening new ones.

### 10a. `NOT` — a third case where the tree wasn't complete

- Steps 6 and 9 gave the tree `AND`/`OR`, but no way to say "not this." `LogicalOperator` couldn't grow a `NOT` case cleanly anyway — `NOT` is unary, and `CompositeFilter`/`GroupNode` are both built around a *list* of children combined pairwise. Forcing `NOT` through that shape (a "group" with exactly one child, meaning something different from every other group) would make the operator enum lie about its own arity.
- Instead, negation gets its own leaf\-like wrapper at both layers, mirroring exactly how the rest of the tree already works: **one type, one child, one job.**

Executable layer — a fourth `Filter` implementation, sitting right alongside `RangeFilter`/`MultiSelectFilter`/`AmenitiesFilter`/`GeoRadiusFilter`/`CompositeFilter`\:

```java
public class NotFilter implements Filter {
    private final Filter child;

    public NotFilter(Filter child) {
        this.child = child;
    }

    @Override
    public boolean isSatisfiedBy(Property property) {
        return !child.isSatisfiedBy(property);
    }
}
```

Descriptive layer — `SearchNode` grows a third permitted subtype:

```java
public sealed interface SearchNode permits CriterionNode, GroupNode, NotNode {
    static SearchNode criterion(FilterCriterion criterion) { return new CriterionNode(criterion); }
    static SearchNode group(LogicalOperator operator, List<SearchNode> children) { return new GroupNode(operator, children); }
    static SearchNode not(SearchNode child) { return new NotNode(child); }
}

public final class NotNode implements SearchNode {
    private final SearchNode child;

    NotNode(SearchNode child) {
        if (child == null) throw new IllegalArgumentException("child must not be null");
        this.child = child;
    }

    public SearchNode getChild() { return child; }
}
```

`SearchRequestCompiler` picks up the new case:

```java
public static Filter compile(SearchNode node, FilterFactory filterFactory) {
    return switch (node) {
        case CriterionNode c -> filterFactory.create(c.getCriterion());
        case GroupNode g -> {
            CompositeFilter composite = new CompositeFilter(g.getOperator());
            g.getChildren().forEach(child -> composite.add(compile(child, filterFactory)));
            yield composite;
        }
        case NotNode n -> new NotFilter(compile(n.getChild(), filterFactory));
    };
}
```

- This is the exhaustiveness guarantee from step 9 actually firing in practice, not just a theoretical claim: adding `NotNode` to `SearchNode`'s `permits` list without adding a matching `case` here would **not compile**. The switch had to be touched, and the compiler is what forced that, not a code review catching a missed case after the fact.
- Usage — negation composes with everything else in the tree, since a `NotNode`'s child can itself be a `GroupNode`, another `NotNode`, or a plain `CriterionNode`\:

```java
// apartments NOT posted by an agent
SearchRequest request = new SearchRequest(
        SearchNode.group(LogicalOperator.AND, List.of(
                SearchNode.criterion(FilterCriterion.multiSelect(Field.PROPERTY_TYPE, Set.of("APARTMENT"))),
                SearchNode.not(SearchNode.criterion(FilterCriterion.multiSelect(Field.POSTED_BY, Set.of("AGENT"))))
        ))
);
platform.search(request);
```

- `FilterCriterion`/`FilterType` themselves were **not** touched — negation is a tree\-structural concept ("invert whatever this subtree decides"), not a new kind of leaf comparison, so it belongs at the `SearchNode`/`Filter` level rather than as a fifth `FilterType`. There's no `NOT_RANGE` or `NOT_MULTI_SELECT` — `NOT` wraps any existing node, including a `GroupNode`, so double negation and "not (A or B)" both fall out for free without any special\-casing.
- `FilterRequestBuilder` (the older, pre\-`SearchNode` path from step 6) was deliberately **left untouched** — it's already a secondary escape hatch, not the primary path, and extending it in parallel would mean maintaining negation support in two places for one feature.

### 10b. Sealing `FilterCriterion`

- Step 4 flagged this directly: the cast\-safety argument in `DefaultFilterFactory` (step 5) held only *by convention* — nothing but discipline stopped a fifth class from implementing `FilterCriterion` outside the sanctioned four and returning a `getType()` that lied about its own class.
- Closed that gap the same way `SearchNode` was already sealed in step 9:

```java
public sealed interface FilterCriterion
        permits RangeCriterion, MultiSelectCriterion, AmenitiesCriterion, GeoRadiusCriterion {
    FilterType getType();
    Field getField();
    // ...static factories, unchanged
}
```

- `sealed` requires every permitted subtype to declare itself `final`, `sealed`, or `non-sealed`. All four concrete criterion classes became `final` — a one\-word change each, since none of them were ever meant to be extended:

```java
public final class RangeCriterion implements FilterCriterion { /* unchanged body */ }
public final class MultiSelectCriterion implements FilterCriterion { /* unchanged body */ }
public final class AmenitiesCriterion implements FilterCriterion { /* unchanged body */ }
public final class GeoRadiusCriterion implements FilterCriterion { /* unchanged body */ }
```

- Nothing else needed to change. `DefaultFilterFactory`'s `switch (criterion.getType())` and its four casts were already written as if this guarantee held — sealing just made that assumption true at the compiler level instead of true by convention. The runtime behavior of every existing query is identical; this is a pure hardening move, verified by re\-running the demo and diffing its output against the pre\-change baseline.
- The asymmetry worth noticing: `SearchNode`'s exhaustiveness (10a) is *exercised* constantly — every time a new node type is added, the compiler forces a matching `case`. `FilterCriterion`'s sealing (10b) is largely *dormant* — it would only ever fire if someone tried to add a fifth criterion type by implementing the interface directly instead of going through the sanctioned pattern (a new concrete class \+ a new `FilterType` case \+ a new `DefaultFilterFactory` branch). Both are worth having; only one of them is likely to ever visibly catch a mistake.

## 11\. Role\-based access — a Protection Proxy in front of the platform

- New requirement, unrelated to filtering: users can be a buyer, a seller, or both, and what they're allowed to *do* — post a property, delete one, buy one, search — depends on which profile they're currently acting as, not just which roles they happen to hold.
- The first shape considered and rejected: a `BuyerProfile`/`SellerProfile` class pair, each with a boolean method per action (`canPost()`, `canBuy()`, ...). This is the exact same mistake as the original `PriceRangeFilter`/`AreaRangeFilter` duplication from step 2 — a new role means a new class, and the permission logic is smeared across N classes instead of living in one place. Rejected for the same reason step 2's fix was: pull the varying part out into data, not new types.
- `Role` — a closed set of profiles a user can hold or act as:

```java
public enum Role {
    BUYER,
    SELLER
}
```

- `Operation` — a closed set of permission\-checkable actions, kept deliberately 1:1 with `PropertyService`'s methods below. If a method doesn't have a matching `Operation`, it isn't enforceable:

```java
public enum Operation {
    POST_PROPERTY,
    DELETE_PROPERTY,
    BUY_PROPERTY,
    SEARCH_PROPERTY
}
```

- `User` — deliberately keeps two different ideas separate instead of collapsing them into one field:
  - `roles`\: every `Role` this user is *eligible* for (can they ever act as a seller?) — fixed at construction.
  - `activeProfile`\: which single `Role` they're operating as *right now* — mutable via `switchProfile()`, but constrained to stay inside `roles`. This is what "depending on what profile the user has selected" actually meant: a buyer/seller user switches hats per request instead of being permanently one or the other.

```java
public class User {
    private final String id;
    private final String name;
    private final Set<Role> roles;
    private Role activeProfile;

    public User(String id, String name, Set<Role> roles, Role activeProfile) {
        if (roles == null || roles.isEmpty()) {
            throw new IllegalArgumentException("roles must not be null or empty");
        }
        this.id = id;
        this.name = name;
        this.roles = Collections.unmodifiableSet(EnumSet.copyOf(roles));
        switchProfile(activeProfile);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public Set<Role> getRoles() { return roles; }
    public Role getActiveProfile() { return activeProfile; }

    public void switchProfile(Role newProfile) {
        if (!roles.contains(newProfile)) {
            throw new IllegalArgumentException(name + " does not hold role " + newProfile);
        }
        this.activeProfile = newProfile;
    }
}
```

- `AccessPolicy` — the Strategy pattern, same shape as `FieldExtractor` (step 3) and `FilterFactory` (step 5): "is this allowed" is pulled into its own one\-method abstraction so the enforcement layer never hardcodes the permission matrix.

```java
public interface AccessPolicy {
    boolean isAllowed(Role role, Operation operation);
}
```

- `RoleAccessPolicy` — the one implementation, a flat `Map<Role, Set<Operation>>` permission matrix. Adding a third role later (`AGENT`, say) means one new map entry here, not a new class:

```java
public class RoleAccessPolicy implements AccessPolicy {
    private final Map<Role, Set<Operation>> permissions;

    public RoleAccessPolicy(Map<Role, Set<Operation>> permissions) {
        this.permissions = new EnumMap<>(permissions);
    }

    @Override
    public boolean isAllowed(Role role, Operation operation) {
        return permissions.getOrDefault(role, Set.of()).contains(operation);
    }

    public static RoleAccessPolicy defaultPolicy() {
        Map<Role, Set<Operation>> matrix = new EnumMap<>(Role.class);
        matrix.put(Role.SELLER, EnumSet.of(Operation.POST_PROPERTY, Operation.DELETE_PROPERTY, Operation.SEARCH_PROPERTY));
        matrix.put(Role.BUYER, EnumSet.of(Operation.BUY_PROPERTY, Operation.SEARCH_PROPERTY));
        return new RoleAccessPolicy(matrix);
    }
}
```

- `PropertyService` — the set of user\-facing, permission\-checkable operations, kept 1:1 with `Operation`. `BookingPlatform` itself stays exactly what it already was — an unaware inventory/search engine that has never heard of a `User`. This interface is a new layer in front of it:

```java
public interface PropertyService {
    void postProperty(Property property, User actor);
    void deleteProperty(String propertyId, User actor);
    void buyProperty(String propertyId, User actor);
    List<Property> search(SearchRequest request, User actor);
}
```

- `DefaultPropertyService` — the real, unprotected implementation. A thin adapter over `BookingPlatform` that doesn't look at `actor` at all; it trusts whoever calls it. `buyProperty` is a stub — `BookingPlatform` has no purchase/transaction concept yet, so this only demonstrates that a permission check would run before any real "buy" logic. Wiring an actual purchase flow is separate, future work.
- `AuthorizingPropertyService` — a **Protection Proxy**, not a Decorator. The distinction matters: a Decorator adds new behavior around a call that always goes through; a Proxy controls whether the call goes through *at all*. Every method here does one thing before ever touching the delegate — check `accessPolicy.isAllowed(actor's active profile, the matching Operation)` — then either throws or forwards unchanged:

```java
public class AuthorizingPropertyService implements PropertyService {
    private final PropertyService delegate;
    private final AccessPolicy accessPolicy;

    public AuthorizingPropertyService(PropertyService delegate, AccessPolicy accessPolicy) {
        this.delegate = delegate;
        this.accessPolicy = accessPolicy;
    }

    @Override
    public void postProperty(Property property, User actor) {
        authorize(actor, Operation.POST_PROPERTY);
        delegate.postProperty(property, actor);
    }

    // deleteProperty / buyProperty / search follow the same shape:
    // authorize(actor, matching Operation) then delegate.

    private void authorize(User actor, Operation operation) {
        if (!accessPolicy.isAllowed(actor.getActiveProfile(), operation)) {
            throw new AccessDeniedException(
                    actor.getName() + " (acting as " + actor.getActiveProfile() + ") is not allowed to " + operation);
        }
    }
}
```

- Callers depend on the `PropertyService` interface, so they can't tell whether they're holding the proxy or the real `DefaultPropertyService` — that's what let this whole layer get added without `BookingPlatform`, `Property`, or any existing filter code changing at all.
- Demoed in `BookingPlatform.main()`\: a seller posts successfully, a buyer is denied `POST_PROPERTY` with `AccessDeniedException`, the buyer buys successfully (their own allowed operation), then switches profile to seller and posts successfully — same `User`, different `activeProfile`, different outcome.
- Known gap, left as a follow\-up rather than silently expanded into scope: this only checks the actor's *role* against the operation ("is any seller allowed to post/delete?"). It does not check *resource ownership* ("is this the seller who posted THIS property?") — `Property` has no `sellerId`/owner reference yet, so today a seller can delete any property, not just their own. Closing that would mean a new `Property.sellerId` field plus a second check inside `deleteProperty` comparing it to `actor.getId()` — a natural next step, not implemented here.

## 12\. Listing lifecycle — the State pattern, and a class boundary worth noticing

- New requirement, again unrelated to filtering: a property needs a life cycle — drafted, published, put under offer when a buyer initiates a purchase, sold, or withdrawn — and each transition has rules (you can't withdraw something mid\-offer; you can't buy something already sold).
- First alternative considered and rejected: a `ListingStatus` enum field plus one method that switches on the current status to decide what's legal. Works, but every new rule grows that one method, and understanding "what can happen from `UNDER_OFFER`" means reading the whole switch instead of one small class. Rejected the same way the enum\-permission\-matrix idea in step 11 was chosen *over* a class\-per\-role — here it's the reverse: a class\-per\-state is the better fit, because the behavior genuinely differs per state, not just per lookup.
- The class\-boundary decision that mattered more than the pattern itself: `Property` (step 7) is immutable on purpose — final fields, defensive copies, safe to share across threads without synchronization. Bolting a mutable lifecycle field onto it would undo that just for this feature. So lifecycle lives on a *new* class, `Listing`, which wraps an immutable `Property` plus the one thing that's actually mutable here. Same split real systems make between an `Order` (what was ordered — doesn't change) and its `OrderStatus` (constantly changing).
- `ListingStatus` — the closed set of states:

```java
public enum ListingStatus {
    DRAFT,
    AVAILABLE,
    UNDER_OFFER,
    SOLD,
    WITHDRAWN
}
```

- `ListingState` — sealed interface, one default method per action, each throwing by default. A concrete state overrides only the actions legal from it — nothing else needs a branch for "not allowed here," the interface already handles that:

```java
public sealed interface ListingState permits Draft, Available, UnderOffer, Sold, Withdrawn {
    ListingStatus getStatus();

    default void publish(Listing listing) { throw invalid("publish"); }
    default void initiatePurchase(Listing listing, User buyer) { throw invalid("initiatePurchase"); }
    default void confirmPurchase(Listing listing) { throw invalid("confirmPurchase"); }
    default void cancelPurchase(Listing listing) { throw invalid("cancelPurchase"); }
    default void withdraw(Listing listing) { throw invalid("withdraw"); }

    private InvalidListingTransitionException invalid(String action) {
        return new InvalidListingTransitionException(action + " is not valid from " + getStatus());
    }
}
```

- Sealed for the same reason `SearchNode`/`FilterCriterion` are (steps 9–10): the compiler guarantees no sixth state exists that nobody accounted for. `permits` names exactly five classes — `Draft`, `Available`, `UnderOffer`, `Sold`, `Withdrawn`.
- Two representative states — `Available` overrides the transitions that move things forward; `UnderOffer` overrides the two that resolve an offer, and *deliberately does not* override `withdraw()`, so a seller can't pull a listing out from under a buyer mid\-purchase without cancelling first:

```java
public final class Available implements ListingState {
    @Override
    public ListingStatus getStatus() { return ListingStatus.AVAILABLE; }

    @Override
    public void initiatePurchase(Listing listing, User buyer) {
        listing.setBuyer(buyer);
        listing.transitionTo(new UnderOffer());
    }

    @Override
    public void withdraw(Listing listing) { listing.transitionTo(new Withdrawn()); }
}

public final class UnderOffer implements ListingState {
    @Override
    public ListingStatus getStatus() { return ListingStatus.UNDER_OFFER; }

    @Override
    public void confirmPurchase(Listing listing) { listing.transitionTo(new Sold()); }

    @Override
    public void cancelPurchase(Listing listing) {
        listing.clearBuyer();
        listing.transitionTo(new Available());
    }
    // no withdraw() override — inherited "invalid" default applies
}
```

- `Sold` and `Withdrawn` are both terminal — neither overrides anything, so every action against them falls through to the interface's throwing default. Cheapest possible way to express "nothing more can happen here."
- `Listing` — the State pattern's *context*. It holds a `Property`, the currently\-active `ListingState`, and (while under offer) the buyer. Every public method is a one\-line delegation; `Listing` itself contains zero transition rules, only the current pointer to which rules apply:

```java
public class Listing {
    private final Property property;
    private ListingState state;
    private User buyer;

    public Listing(Property property) {
        this.property = property;
        this.state = new Draft();
    }

    public synchronized ListingStatus getStatus() { return state.getStatus(); }
    public synchronized void publish() { state.publish(this); }
    public synchronized void initiatePurchase(User buyer) { state.initiatePurchase(this, buyer); }
    public synchronized void confirmPurchase() { state.confirmPurchase(this); }
    public synchronized void cancelPurchase() { state.cancelPurchase(this); }
    public synchronized void withdraw() { state.withdraw(this); }

    // package-private — only ListingState implementations may call these
    void transitionTo(ListingState newState) { this.state = newState; }
    void setBuyer(User buyer) { this.buyer = buyer; }
    void clearBuyer() { this.buyer = null; }
}
```

- Every method is `synchronized`\: two threads racing to buy the same listing must not both succeed — the check\-and\-transition has to be atomic, not just each field write in isolation. `transitionTo`/`setBuyer`/`clearBuyer` are package\-private, so only the state classes (same package) can force a transition; nothing outside can reach in and mutate lifecycle directly.
- `ListingRegistry` — a small `Map<String, Listing>` (property id to its `Listing`), deliberately kept *separate* from `BookingPlatform`. This was the other seam worth protecting: `BookingPlatform`'s `List<Property>` and its whole filter engine (steps 1–10) never needed to learn that lifecycle exists. Nothing in `filter`/`criterion`/`field` changed for this feature — the registry is a new, parallel structure the service layer consults, not something wired into `CompositeFilter` or `FieldExtractor`.
- Wiring into `PropertyService` (step 11): `postProperty` now registers a `Listing` starting in `DRAFT` — not searchable until a new `publishListing` call moves it to `AVAILABLE`. `deleteProperty` now calls `listing.withdraw()` *before* removing it from inventory, so deleting a listing that's `UNDER_OFFER` or `SOLD` correctly fails — a real rule that didn't exist when this was unconditional `properties.removeIf(...)`. `buyProperty` now calls `listing.initiatePurchase(actor)` instead of a println stub; two new operations, `confirmPurchase` and `cancelPurchase`, resolve the offer either way. `search` filters `BookingPlatform`'s raw results down to `AVAILABLE` listings — a property with no registered `Listing` at all (e.g. anything added directly through `BookingPlatform.addProperty`, bypassing the service) is treated as visible by default, which is what keeps every pre\-existing filter\-engine demo property (steps 1–10) showing up unchanged.
- Three new `Operation` values (`PUBLISH_LISTING`, `CONFIRM_PURCHASE`, `CANCEL_PURCHASE`) slot into the permission matrix from step 11 exactly the way step 11 said a new value would: one line each, no new classes — `SELLER` gets `PUBLISH_LISTING`, `BUYER` gets `CONFIRM_PURCHASE`/`CANCEL_PURCHASE`.
- Demoed end to end in `BookingPlatform.main()`\: a listing posted (`DRAFT`, invisible to search) → published (`AVAILABLE`, now visible) → bought (`UNDER_OFFER`) → an attempted withdraw correctly rejected with `InvalidListingTransitionException` → confirmed (`SOLD`) → an attempted cancel correctly rejected. A second listing walks the other branch: bought, then cancelled back to `AVAILABLE`.
- Known gap, surfaced by the demo itself rather than hidden: the same user posted one of the demo listings as a seller and then bought it as a buyer, and it succeeded — because `AuthorizingPropertyService` (step 11) checks *role*, not *resource ownership*. That gap was already flagged in step 11; this step just makes it concrete. Still deferred, still would need `Property.sellerId` plus an ownership comparison to close.

## 13\. Property alerts — the Observer pattern, reusing step 9's compiler for free

- New requirement: a buyer saves a search, and gets notified when a new listing that matches it goes live. The trigger is precise — `publishListing` (step 12) is the one and only moment a listing becomes visible, so it's the one and only moment worth watching.
- The two Observer\-pattern roles split cleanly, and land in different shapes for a reason already established by this codebase: the *listener* is a proper interface, because multiple implementations are expected by design (today a saved\-search alert, plausibly an analytics hook or a webhook forwarder later) — same reasoning as `Filter`/`AccessPolicy`/`ListingState`. The *subject* is a plain concrete class, because exactly one implementation was ever going to exist — same reasoning as `BookingPlatform`/`FilterEngine` staying un\-interfaced.

```java
public interface PropertyPublishedListener {
    void onPropertyPublished(Property property);
}
```

- `PropertyAlertService` — the subject. Holds listeners, notifies them, nothing else:

```java
public class PropertyAlertService {
    private final List<PropertyPublishedListener> listeners = new CopyOnWriteArrayList<>();

    public void subscribe(PropertyPublishedListener listener) { listeners.add(listener); }
    public void unsubscribe(PropertyPublishedListener listener) { listeners.remove(listener); }

    public void publish(Property property) {
        for (PropertyPublishedListener listener : listeners) {
            try {
                listener.onPropertyPublished(property);
            } catch (RuntimeException e) {
                System.out.println("Listener failed, continuing: " + e.getMessage());
            }
        }
    }
}
```

- `CopyOnWriteArrayList` again — third time this exact call has been made (step 7's `BookingPlatform.properties`, unchanged since). `publish()` iterates the listener list on every single listing that goes live; `subscribe`/`unsubscribe` are comparatively rare. Same read\-heavy/write\-light shape, same fix.
- The per\-listener `try`/`catch` is a small thing worth stating on purpose rather than leaving implicit: one broken subscriber throwing shouldn't stop every other subscriber from hearing about a new listing. Isolating failures per\-observer is a standard real\-world Observer concern that's easy to skip in a first pass.
- The part of this feature that needed the least new code: **matching an already\-saved search against a newly\-published property is exactly what step 9's `SearchRequestCompiler` and step 2's `Filter.isSatisfiedBy(Property)` already do.** A saved search is a `SearchRequest`; `SearchRequestCompiler.compile(request.getRoot(), filterFactory)` turns it into a `Filter`; that `Filter` already knows how to answer "does this one property satisfy me." Nothing new was written to do the actual matching — the concrete listener just holds a pre\-compiled `Filter` and asks it:

```java
public class SavedSearchAlertListener implements PropertyPublishedListener {
    private final User buyer;
    private final Filter matcher;

    public SavedSearchAlertListener(User buyer, Filter matcher) {
        this.buyer = buyer;
        this.matcher = matcher;
    }

    @Override
    public void onPropertyPublished(Property property) {
        if (matcher.isSatisfiedBy(property)) {
            System.out.println(buyer.getName() + " ALERT: new listing matches your saved search — " + property);
        }
    }
}
```

- Deliberately takes an already\-compiled `Filter`, not a raw `SearchRequest` \+ `FilterFactory` — compiling is the subscribing caller's job, done once at subscription time. `SavedSearchAlertListener` itself has no idea `SearchRequestCompiler` exists; it only knows `Filter.isSatisfiedBy`, a two\-step\-old interface from step 2.
- Wiring: `DefaultPropertyService` (step 11) takes `PropertyAlertService` as a third constructor dependency, alongside `BookingPlatform` and `ListingRegistry`. `publishListing` now does `listing.publish(); alertService.publish(listing.getProperty());` — publish the listing, then tell anyone watching.
- A scope boundary worth stating out loud: subscribing to alerts does **not** go through `AuthorizingPropertyService` (step 11). That proxy protects property CRUD and lifecycle operations — who can post, delete, buy, confirm, cancel. Watching for new listings is a different, self\-service concern; any buyer can subscribe to any saved search without a role check. Not every new capability has to route through an existing gate just because one exists.
- Demoed in `BookingPlatform.main()`\: a buyer subscribes with a saved search (2BHK, under 40L, in Andheri); a matching listing published → alert fires with the property; a non\-matching listing published → nothing; buyer unsubscribes → a second matching listing published → nothing, confirming `unsubscribe` actually removes the listener rather than just leaving it silently allowed through some other check.

## 14\. Dynamic pricing — the Decorator pattern, and the contrast with Proxy that's worth stating out loud

- New requirement: a property's quoted price isn't just its listed price — a seasonal surcharge, a loyalty discount, and tax can all apply on top, in varying combinations depending on the buyer and the moment.
- `Property.price` (step 7) stays exactly what it's always been: the seller's listed price, immutable. None of this pricing logic touches `Property` — same instinct as step 12's `Listing`, applied again: a concern that varies per\-call shouldn't live on the immutable model.
- `PriceCalculator` — the Decorator pattern's Component. One method, deliberately the same shape whether the caller is holding the bare calculator or a whole stack wrapped around it:

```java
public interface PriceCalculator {
    double calculate(Property property);
}

public class BasePriceCalculator implements PriceCalculator {
    @Override
    public double calculate(Property property) { return property.getPrice(); }
}
```

- `PriceCalculatorDecorator` — the abstract Decorator, holding the wrapped calculator so every concrete decorator doesn't redeclare the same field and constructor:

```java
public abstract class PriceCalculatorDecorator implements PriceCalculator {
    protected final PriceCalculator delegate;

    protected PriceCalculatorDecorator(PriceCalculator delegate) {
        this.delegate = delegate;
    }
}
```

- **The contrast worth stating explicitly, now that both patterns exist in this codebase:** `AuthorizingPropertyService` (step 11) is a Protection Proxy — it controls *whether* a call goes through at all, and can refuse it outright. Every decorator here *always* calls `delegate.calculate(...)` — none of them can refuse to price a property, they can only transform the number that comes back. Proxy gates access; Decorator adds behavior around access that's already granted. Having built one of each rather than just describing the difference is the whole value of doing this as a second pattern instead of stopping at Proxy.
- Three concrete decorators — two percentage\-based, one deliberately flat:

```java
public class SeasonalSurchargeDecorator extends PriceCalculatorDecorator {
    private final double flatSurcharge;

    public SeasonalSurchargeDecorator(PriceCalculator delegate, double flatSurcharge) {
        super(delegate);
        this.flatSurcharge = flatSurcharge;
    }

    @Override
    public double calculate(Property property) {
        return delegate.calculate(property) + flatSurcharge;
    }
}

public class LoyaltyDiscountDecorator extends PriceCalculatorDecorator {
    private final double discountRate;
    // constructor mirrors SeasonalSurchargeDecorator's

    @Override
    public double calculate(Property property) {
        double amount = delegate.calculate(property);
        return amount - (amount * discountRate);
    }
}

public class TaxDecorator extends PriceCalculatorDecorator {
    private final double taxRate;
    // constructor mirrors the others

    @Override
    public double calculate(Property property) {
        double amount = delegate.calculate(property);
        return amount + (amount * taxRate);
    }
}
```

- `SeasonalSurchargeDecorator` is deliberately **additive** (a flat amount) while the other two are **multiplicative** (a percentage). That mix is not an accident: two purely percentage\-based decorators commute — `(1 - discount) * (1 + tax)` equals `(1 + tax) * (1 - discount)` regardless of order — which would hide the one thing about Decorator most worth learning here. Mixing a flat and a percentage adjustment makes order genuinely change the result:
  - Surcharge, then loyalty discount: `(base + flatSurcharge) * (1 - discountRate)`
  - Loyalty discount, then surcharge: `(base * (1 - discountRate)) + flatSurcharge`
  - On P8 (base 3,300,000, flat surcharge 100,000, 10% discount): **3,060,000** vs **3,070,000** — same two decorators, same two rates, different totals purely from stacking order.
- That asymmetry is the risk side of Decorator's flexibility, worth stating as clearly as the benefit: nothing about the pattern *prevents* a nonsensical composition. Contrast with `ListingState` (step 12), which makes an invalid sequence a compile\-time\-guaranteed runtime exception (`InvalidListingTransitionException`) — Decorator offers no equivalent guardrail. Composing decorators in a sensible order is a discipline the caller has to bring themselves; the pattern will happily calculate a wrong\-but\-plausible\-looking number if you get it backwards.
- The recommended, stated\-on\-purpose order for a full quote: surcharge adjusts the base cost first, loyalty discount applies to that subtotal, tax is computed last on top of everything else:

```java
PriceCalculator fullQuote = new TaxDecorator(
        new LoyaltyDiscountDecorator(
                new SeasonalSurchargeDecorator(basePrice, flatSurcharge), loyaltyDiscountRate),
        taxRate);
```

- One `BasePriceCalculator` instance is reused across every stack in the demo — it's stateless, so nothing stops five different decorator chains from sharing the same base rather than each constructing their own.
- Demoed in `BookingPlatform.main()`\: base price printed unadorned, then the same two decorators applied in both orders side by side (proving the order\-sensitivity concretely, not just asserting it), then the full recommended stack producing one final quote.

## 15\. Listing validation — Chain of Responsibility, and a bug the verification step actually caught

- New requirement: before a listing can go live, it should pass a series of independent checks — is the price sane, are the required fields present, is it a likely duplicate of something already listed. Each check is its own concern; a sloppy price shouldn't be tangled up with duplicate\-detection logic in one giant validation method.
- `ListingValidationHandler` — the abstract handler. `handle()` is a `final` template method: validate this link, then forward to the next one, but only if validating didn't throw. Concrete subclasses only ever implement `validate()` — they never call the next link themselves, so the actual chaining logic exists in exactly one place:

```java
public abstract class ListingValidationHandler {
    private ListingValidationHandler next;

    public ListingValidationHandler setNext(ListingValidationHandler next) {
        this.next = next;
        return next;
    }

    public final void handle(Property property) {
        validate(property);
        if (next != null) {
            next.handle(property);
        }
    }

    protected abstract void validate(Property property);
}
```

- There's no pass/fail return value to check after calling `handle()` — a handler that rejects the property simply throws `ListingValidationException` and the chain stops right there; a handler that's satisfied just returns normally and `handle()`'s template forwards to the next link. Simpler than threading a boolean or a result object through every handler.
- Three concrete links, chained once at construction time:

```java
public class PriceSanityHandler extends ListingValidationHandler {
    @Override
    protected void validate(Property property) {
        if (property.getPrice() <= 0) { throw new ListingValidationException(/* ... */); }
        if (property.getPrice() > MAX_REASONABLE_PRICE) { throw new ListingValidationException(/* ... */); }
    }
}

public class RequiredFieldsHandler extends ListingValidationHandler {
    @Override
    protected void validate(Property property) {
        // id, bhk, propertyType, furnishing, location must all be present
    }
}

public class DuplicateListingHandler extends ListingValidationHandler {
    private final ListingRegistry listingRegistry;
    // scans every OTHER registered listing for a location/price/bhk match
}
```

- `DuplicateListingHandler` is the one link that needs a live dependency — `ListingRegistry` — so the chain can't be fully static; it's assembled once, in `DefaultPropertyService`'s constructor, and reused for every `publishListing` call rather than rebuilt each time:

```java
private static ListingValidationHandler buildValidationChain(ListingRegistry listingRegistry) {
    ListingValidationHandler priceCheck = new PriceSanityHandler();
    priceCheck.setNext(new RequiredFieldsHandler()).setNext(new DuplicateListingHandler(listingRegistry));
    return priceCheck;
}
```

- Wired into `publishListing` *before* the `ListingState` transition (step 12) even runs: `validationChain.handle(listing.getProperty()); listing.publish();` — a rejected property never reaches `Draft.publish()`, so it simply stays in `DRAFT` rather than needing its own rejected\-but\-not\-quite\-draft status.
- **A bug the verification step actually caught, worth keeping in the record rather than quietly fixing off\-screen:** the first version of `DuplicateListingHandler` compared the property being validated against every registered listing's location — but `postProperty` registers a `Listing` in `DRAFT` *before* any validation ever runs (validation only happens at `publishListing` time). That means a property that already failed `RequiredFieldsHandler` earlier (missing location) can still be sitting in the registry with a `null` location. Publishing a *different*, valid property then walked into `DuplicateListingHandler`, iterated over that earlier bad listing, and threw a `NullPointerException` calling `.getLat()` on a null `GeoPoint`. Running the demo before touching the real project — the verification step this whole session has followed since step 1 — is what surfaced this, not a code review. Fixed by skipping any comparison listing whose own location is null:

```java
if (other.getId().equals(property.getId()) || other.getLocation() == null) {
    continue;
}
```

- The lesson generalizes past this one bug: a handler that only reads the property currently being validated is safe by construction — `PriceSanityHandler` and `RequiredFieldsHandler` can't be broken by any other listing's bad data. A handler like `DuplicateListingHandler`, which reaches into *other* stored state, inherits whatever invariants (or lack of them) that other state actually has — and "every registered `Listing` has a fully valid `Property`" was never actually true, since registration and validation are deliberately two different moments (step 12's `DRAFT` state exists precisely so a listing can be registered before it's ready to go live).
- Also compares `GeoPoint` by `getLat()`/`getLng()` value rather than by reference — `GeoPoint` (step 1) has no `equals()` override, so relying on object identity would happen to work in this demo, which reuses shared `bandra`/`andheri`/`powai` instances across many properties, but wouldn't hold for two independently constructed points carrying the same coordinates. Worth calling out as a place where the demo's convenience could have masked a real correctness gap if the comparison had been written the lazy way.
- Demoed in `BookingPlatform.main()`\: a negative\-price listing rejected by the first link, a location\-less listing rejected by the second (with the first link never re\-run — each handler only runs once per `handle()` call, and rejection at any link stops the rest), a price/bhk/location duplicate of an already\-published listing rejected by the third, and a fully valid listing passing all three and publishing normally.

## 16\. Paying for a purchase — the Adapter pattern, for a mismatch this codebase doesn't control

- New requirement: `confirmPurchase` (step 12) should actually charge the buyer before letting a listing's `ListingState` move to `SOLD`. The twist that makes this a different pattern from everything else so far: the thing being charged through is a **third\-party payment SDK**, not code this project owns.
- Every pattern up to this point has been about organizing *our own* code — Strategy swaps our own algorithms, State swaps our own behavior per lifecycle stage. Adapter is specifically for the case where the incompatible shape belongs to someone else, and changing it isn't an option. That distinction is worth stating plainly, because it also separates Adapter from Strategy on the one side ("my own interchangeable code" vs. "someone else's fixed shape") and from Proxy/Decorator on the other (those wrap something that already speaks our language; Adapter's whole job is translating something that doesn't).
- `PaymentGateway` — the Target interface, the contract our code depends on:

```java
public interface PaymentGateway {
    PaymentResult charge(User payer, double amountInRupees);
}
```

- Two pretend vendor SDKs, deliberately shaped the way real ones actually look, not simplified for convenience:

```java
// FastPay: paise instead of rupees, and its own response type
public class FastPaySdk {
    public FastPayResponse makePayment(String customerId, long amountInPaise) { /* ... */ }
}
public class FastPayResponse {
    public final boolean approved;
    public final String referenceCode;
    public final String errorCode;
}

// LegacyBank: a status code instead of an exception/boolean, and the transaction id has to be
// fetched with a SEPARATE call afterward instead of being returned directly
public class LegacyBankGatewaySdk {
    public int submitTransaction(String accountHolderName, double amountInRupees) { /* returns 0 or an error code */ }
    public String getLastTransactionId() { /* only meaningful right after a submitTransaction() that returned 0 */ }
}
```

- One Adapter per vendor, each absorbing its own vendor's specific ugliness so nothing downstream ever sees it:

```java
public class FastPayAdapter implements PaymentGateway {
    private final FastPaySdk sdk;

    @Override
    public PaymentResult charge(User payer, double amountInRupees) {
        long amountInPaise = Math.round(amountInRupees * 100);
        FastPayResponse response = sdk.makePayment(payer.getId(), amountInPaise);
        return response.approved
                ? PaymentResult.success(response.referenceCode)
                : PaymentResult.failure("FastPay declined: " + response.errorCode);
    }
}

public class LegacyBankGatewayAdapter implements PaymentGateway {
    private final LegacyBankGatewaySdk sdk;

    @Override
    public PaymentResult charge(User payer, double amountInRupees) {
        int statusCode = sdk.submitTransaction(payer.getName(), amountInRupees);
        return statusCode != 0
                ? PaymentResult.failure("LegacyBank error code " + statusCode)
                : PaymentResult.success(sdk.getLastTransactionId());
    }
}
```

- `FastPayAdapter` absorbs a **unit conversion** (rupees to paise); `LegacyBankGatewayAdapter` absorbs a **protocol shape** (status code plus a second, stateful call to retrieve the id). Two different kinds of mismatch, same pattern, same fix: whichever adapter a caller holds, `charge()` takes a `User` and a rupee amount and returns one `PaymentResult` — nobody outside the two adapter classes ever sees a `FastPayResponse`, a paise value, a status code, or `getLastTransactionId()`.
- Wired into `confirmPurchase` (step 12) as one more constructor dependency on `DefaultPropertyService`, alongside `ListingRegistry`, `PropertyAlertService`, and the validation chain (step 15):

```java
PaymentResult result = paymentGateway.charge(actor, listing.getProperty().getPrice());
if (!result.isSuccess()) {
    throw new PaymentFailedException(result.getFailureReason());
}
listing.confirmPurchase();
```

- A failed charge means `listing.confirmPurchase()` never runs — the `ListingState` transition to `SOLD` simply doesn't happen, and the listing stays `UNDER_OFFER`, exactly as if the buyer had chosen to cancel instead. No new `ListingState` was needed for "payment failed" — it's just "the state didn't move forward," which `Listing` already supported by construction.
- No changes were needed to `PropertyService`, `Operation`, or `AuthorizingPropertyService` (step 11) for any of this — `confirmPurchase` already existed as a permission\-checked operation; charging the buyer is purely an internal detail of what that method does now, invisible to the authorization layer.
- Demoed in `BookingPlatform.main()`\: `FastPayAdapter` and `LegacyBankGatewayAdapter` called directly, side by side, for the same amount — proving they're truly interchangeable through `PaymentGateway` despite completely different vendor internals; then the same two adapters charging an amount that exceeds FastPay's pretend transaction limit but not LegacyBank's, so one succeeds and the other doesn't for reasons specific to that one vendor; then a full `confirmPurchase` that succeeds because the price is under the limit, and one that throws `PaymentFailedException` because it isn't — with the listing verified to still be `UNDER_OFFER`, not `SOLD`, afterward.

## 17\. Undoable listing actions — the Command pattern, and the honesty of an undo() that sometimes can't

- New requirement: let a caller execute a listing action and, if they change their mind, undo it — without `CommandHistory` (the thing tracking "what happened") needing to know anything about *which* action it's holding. That's the Command pattern in one sentence: turn a method call into an object, so it can be stored, passed around, and reversed uniformly.
- This is deliberately layered *on top of* `PropertyService`, not inside it: `ListingCommand` implementations are thin wrappers that each hold a `PropertyService`, a property id, and an actor, and call exactly one existing method. No changes were needed to `PropertyService`, `Operation`, or `AuthorizingPropertyService` — commands are an orchestration concern for whoever's calling the service, not a new permission surface.

```java
public interface ListingCommand {
    void execute();
    void undo();
}
```

- Four commands wrap the four `PropertyService` methods that actually move a listing through `ListingState` (step 12): `publishListing`, `buyProperty`, `confirmPurchase`, `cancelPurchase`. `postProperty` and `deleteProperty` were deliberately left out — creating and permanently removing a listing aren't "undo a recent action" candidates the way a lifecycle transition is.
- The interesting decision is what happens when `undo()` is asked of an action `ListingState` doesn't actually support reversing. Two of the four are genuine inverses of each other:

```java
public class BuyPropertyCommand implements ListingCommand {
    private final PropertyService service;
    private final String propertyId;
    private final User actor;

    @Override
    public void execute() { service.buyProperty(propertyId, actor); }

    @Override
    public void undo() { service.cancelPurchase(propertyId, actor); }
}

public class CancelPurchaseCommand implements ListingCommand {
    // ... same fields ...
    @Override
    public void execute() { service.cancelPurchase(propertyId, actor); }

    @Override
    public void undo() { service.buyProperty(propertyId, actor); }
}
```

- The other two aren't reversible, and pretending otherwise would be a lie the domain doesn't support: `publishListing` moves `DRAFT` to `AVAILABLE`, and `ListingState` has never defined a path back — the same one\-way design already used for `Withdrawn` in step 12. `confirmPurchase` moves to `SOLD` (terminal) and, since step 16, has already charged the buyer through `PaymentGateway` — undoing it would mean issuing a refund, which nothing in this codebase models. Both commands' `undo()` throws a new `CommandNotUndoableException` instead of silently no\-op\-ing or leaving the listing in a state the rest of the code doesn't expect:

```java
public class PublishListingCommand implements ListingCommand {
    // ...
    @Override
    public void undo() {
        throw new CommandNotUndoableException(
                "Cannot un-publish " + propertyId + " -- no AVAILABLE -> DRAFT transition exists");
    }
}
```

`ConfirmPurchaseCommand.undo()` throws the same way, citing the missing refund flow instead of the missing state transition.

- `CommandHistory` is the Invoker — it runs a command and pushes it onto a stack, and `undoLast()` pops and reverses the most recent one:

```java
public class CommandHistory {
    private final Deque<ListingCommand> history = new ArrayDeque<>();

    public void execute(ListingCommand command) {
        command.execute();
        history.push(command);
    }

    public void undoLast() {
        if (history.isEmpty()) {
            throw new IllegalStateException("Nothing to undo");
        }
        ListingCommand last = history.peek();
        last.undo();
        history.pop();
    }
}
```

- `undoLast()` peeks before it pops: the command only comes off the stack once `undo()` returns without throwing. If the most recent command was a `PublishListingCommand`, `undoLast()` throws `CommandNotUndoableException` and the command stays on top of the stack — the record of what actually happened isn't discarded just because reversing it turned out to be impossible.
- One more thing the demo makes visible: undo isn't a magic "restore the previous snapshot" — it's just re\-invoking the inverse operation, which means it's still subject to `ListingState`'s own rules. After `BuyPropertyCommand.undo()` puts a listing back to `AVAILABLE`, calling a fresh `CancelPurchaseCommand.execute()` on it fails with the same `InvalidListingTransitionException` step 12 already defined for "cancel a purchase that was never in progress" — Command doesn't bypass State, it just calls through it.
- Demoed in `BookingPlatform.main()`\: a listing bought then undone via `CommandHistory` (verified back to `AVAILABLE`), a direct re\-cancel attempt on that same listing failing for the reason above, and a separate listing published then an `undoLast()` on it caught as `CommandNotUndoableException` — the boundary of the pattern shown failing loudly rather than glossed over.

## 18\. Domestic vs NRI market bundles — the Abstract Factory pattern, closing the loop on three existing products

- New requirement: nothing new to demonstrate on its own, but a real gap in how three earlier steps got wired together. `DefaultPropertyService` picked a `PaymentGateway` (Adapter, step 16) and separately built its own `ListingValidationHandler` chain (Chain of Responsibility, step 15) — two independent choices with nothing tying them together. A caller could pass a payment gateway with a *low* transaction limit alongside a validation chain with a *high* price cap, and the mismatch would only surface at charge time, as a confusing runtime failure instead of a configuration error caught up front.
- Abstract Factory's whole purpose is exactly this: produce a **family of related products together**, so the caller picks one factory and gets a bundle guaranteed to agree with itself, instead of assembling the pieces by hand.

```java
public interface MarketServiceFactory {
    PaymentGateway createPaymentGateway();
    ListingValidationHandler createValidationChain(ListingRegistry listingRegistry);
    PriceCalculator createPriceCalculator();
}
```

- The family has three products, all reusing patterns already in this codebase rather than inventing new ones: `PaymentGateway` (Adapter, step 16), `ListingValidationHandler` (Chain of Responsibility, step 15), and — added to the bundle at the same time — `PriceCalculator` (Decorator, step 14), which until now had never actually been wired into `DefaultPropertyService` at all; it only existed as a standalone demo in `BookingPlatform.main()`.
- Two concrete factories, each a believable bundle:

```java
public class DomesticMarketFactory implements MarketServiceFactory {
    private static final double DOMESTIC_PRICE_CAP = 100_000_000; // 10 crore — same as before this step
    private static final double DOMESTIC_TAX_RATE = 0.05;

    @Override
    public PaymentGateway createPaymentGateway() {
        return new FastPayAdapter(new FastPaySdk());
    }

    @Override
    public ListingValidationHandler createValidationChain(ListingRegistry listingRegistry) {
        ListingValidationHandler priceCheck = new PriceSanityHandler(DOMESTIC_PRICE_CAP);
        priceCheck.setNext(new RequiredFieldsHandler()).setNext(new DuplicateListingHandler(listingRegistry));
        return priceCheck;
    }

    @Override
    public PriceCalculator createPriceCalculator() {
        return new TaxDecorator(new BasePriceCalculator(), DOMESTIC_TAX_RATE);
    }
}

public class NriMarketFactory implements MarketServiceFactory {
    private static final double NRI_PRICE_CAP = 500_000_000; // 50 crore
    private static final double NRI_TAX_RATE = 0.12;

    @Override
    public PaymentGateway createPaymentGateway() {
        return new LegacyBankGatewayAdapter(new LegacyBankGatewaySdk());
    }

    @Override
    public ListingValidationHandler createValidationChain(ListingRegistry listingRegistry) {
        ListingValidationHandler priceCheck = new PriceSanityHandler(NRI_PRICE_CAP);
        priceCheck.setNext(new RequiredFieldsHandler()).setNext(new DuplicateListingHandler(listingRegistry));
        return priceCheck;
    }

    @Override
    public PriceCalculator createPriceCalculator() {
        return new TaxDecorator(new BasePriceCalculator(), NRI_TAX_RATE);
    }
}
```

- `PriceSanityHandler` (step 15) picked up a small, backward\-compatible change to make this possible: the price cap moved from a hardcoded constant to a constructor parameter, with a no\-arg constructor keeping the original 10\-crore default. A configuration knob, not new validation logic.
- `DefaultPropertyService`'s constructor changed from taking a `PaymentGateway` directly to taking a `MarketServiceFactory`, and derives all three products from it:

```java
public DefaultPropertyService(BookingPlatform platform, ListingRegistry listingRegistry,
        PropertyAlertService alertService, MarketServiceFactory marketFactory) {
    this.platform = platform;
    this.listingRegistry = listingRegistry;
    this.alertService = alertService;
    this.validationChain = marketFactory.createValidationChain(listingRegistry);
    this.paymentGateway = marketFactory.createPaymentGateway();
    this.priceCalculator = marketFactory.createPriceCalculator();
}
```

- Giving `PriceCalculator` an actual caller meant adding one new operation, `quotePrice`, following the exact precedent step 12 set for every prior addition: a new `PropertyService` method, a matching `Operation.QUOTE_PRICE`, a pass\-through in `AuthorizingPropertyService`, and an entry in `RoleAccessPolicy.defaultPolicy()` for both roles (anyone who can see a listing can price it):

```java
@Override
public double quotePrice(String propertyId, User actor) {
    Property property = listingRegistry.get(propertyId).getProperty();
    return priceCalculator.calculate(property);
}
```

- Demoed in `BookingPlatform.main()`\: the exact same listing, posted once, run through two different `DefaultPropertyService` instances built from `DomesticMarketFactory` and `NriMarketFactory`. Publishing it through the domestic service fails — `ListingValidationException`, over the 10\-crore cap — while publishing the identical listing through the NRI service succeeds under its 50\-crore cap; `quotePrice` through each then returns a different total (5% vs. 12% tax) for the same base price. One factory choice changed the price cap, the payment gateway, and the tax rate together, consistently, instead of three settings that a caller could have mismatched by hand.
- No changes were needed to `PropertyService`'s existing methods, `Operation`'s existing values, or `BookingPlatform`/`Property` beyond the one new operation — the same kind of clean layering every pattern since step 11 has kept: this is still a permission\-checked, market\-aware policy layer in front of an inventory engine that has no idea any of this exists.

## 19\. Replacing the `main()` walkthrough with a real JUnit test suite

- Not a design pattern — a housekeeping step the codebase had earned. By step 18, `BookingPlatform.main()` had grown to eleven "Examples," 400\+ lines, each one printing state to stdout for a human to eyeball. That's exactly what a test suite is for, and this project already had JUnit 5 on its classpath (`junit-jupiter-api`/`-params` were in `pom.xml` from the start, just never exercised) — `main()` had been standing in for real tests the whole time.
- `main()` is gone. `BookingPlatform` itself is unchanged except for losing that method — `addProperty`, `removeProperty`, `search`, `inventorySize` are exactly what they were.
- One JUnit test class per pattern area, mirroring the package structure, each replacing one or two of the old numbered Examples:

| Example(s) | Test class | Pattern |
| --- | --- | --- |
| 1–3 | `filter.FilterEngineTest` | Strategy \+ Composite \+ Builder (steps 1–10) |
| 4 | `security.RoleBasedAccessTest` | Protection Proxy (step 11) |
| 5 | `listing.ListingLifecycleTest` | State (step 12) |
| 6 | `alert.PropertyAlertServiceTest` | Observer (step 13) |
| 7 | `pricing.PriceCalculatorDecoratorTest` | Decorator (step 14) |
| 8 | `validation.ListingValidationChainTest` | Chain of Responsibility (step 15) |
| 9 | `payment.PaymentGatewayAdapterTest` | Adapter (step 16) |
| 10 | `command.ListingCommandTest` | Command (step 17) |
| 11 | `market.MarketServiceFactoryTest` | Abstract Factory (step 18) |

- The most important structural change isn't the annotations — it's that every test method now gets its **own, from\-scratch fixtures**. `main()`'s eleven Examples ran in sequence against one long\-lived `BookingPlatform`/`ListingRegistry`/set of Users — Example 8's P11–P14 existing, in a specific state, was load\-bearing for Example 9 even reaching the code it wanted to exercise. That's exactly the kind of order\-dependent, shared\-mutable\-state design a test suite is supposed to eliminate: a `support.TestFixtures` helper class now supplies a fresh `BookingPlatform`, geo points, and two Users on demand, and every `@BeforeEach` builds its own `ListingRegistry`/`PropertyAlertService`/`PropertyService` from nothing.
- Two places needed a small testing trick because their real work is a side effect rather than a return value, which the tests treat as worth calling out rather than working around quietly:
  - `PropertyAlertServiceTest` exercises `PropertyAlertService` itself (subscribe/publish/unsubscribe, and a broken listener not stopping the others) with a trivial recording `PropertyPublishedListener` that just appends to a list — no `Filter` involved at all. `SavedSearchAlertListener`'s own behavior is separately verified by redirecting `System.out` around a real `onPropertyPublished` call, since printing is literally its only observable effect.
  - Nothing else needed that trick — every other pattern's effect is a state change (`ListingStatus`), a return value (`PaymentResult`, a quoted price), or a thrown exception, all of which `assertEquals`/`assertThrows` check directly.
- The Chain of Responsibility tests (`ListingValidationChainTest`) specifically keep the NPE regression from step 15 as its own test — `duplicateCheckSkipsAPreviouslyRegisteredListingWithNoLocationInsteadOfThrowingNpe` — registering an invalid, null\-location listing first and then confirming a later, unrelated listing still publishes cleanly. A bug the verification workflow caught once, by hand, now has a permanent regression test instead of only a paragraph in this document.
- `pom.xml` needed one addition: `junit-jupiter-engine` (test scope), alongside the `junit-jupiter-api`/`-params` that were already there. `junit-jupiter-api` alone declares the annotations; the engine is what Surefire actually needs on the classpath to discover and run them. `maven-surefire-plugin`'s version was already pinned in `pluginManagement`, and 3.3.0 auto\-detects the JUnit Platform provider once the engine dependency is present — no explicit provider configuration needed.

## 20\. A smoke test alongside the unit tests — trading isolation for a real end\-to\-end pass

Step 19 replaced `main()`'s eleven examples with nine granular JUnit test classes, one per pattern area, each building its own fresh fixtures in `@BeforeEach`. That's the right default for unit tests: each test is independent, order doesn't matter, and a failure points at exactly one thing. But it also throws away something the old `main()` had almost by accident — a single continuous run through every pattern in sequence, sharing one `BookingPlatform`, one `ListingRegistry`, one seller and buyer, the way the system actually gets used. Isolated tests can't catch a bug that only shows up when step 6's alert side effect leaks into step 7's pricing call, or when a listing published in step 5 is still sitting in `AVAILABLE` when step 8's validation chain runs against it. That class of bug is exactly what a smoke test is for, and it's what the user asked for explicitly: "smoke tests — similar to what we were earlier doing in the main class."

The mechanism is three JUnit annotations working together, none of which the granular tests needed:

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class SmokeTest {
    private BookingPlatform platform;
    private FilterFactory filterFactory;
    private ListingRegistry listingRegistry;
    private PropertyAlertService alertService;
    private PropertyService propertyService;
    private User seller;
    private User buyer;

    @Test @Order(1) void step01_inventoryAndAndQuery() { ... }
    @Test @Order(2) void step02_orOfGroupsQuery() { ... }
    ...
    @Test @Order(11) void step11_abstractFactoryMarkets() { ... }
}
```

`@TestInstance(PER_CLASS)` tells JUnit to build the test class exactly once and reuse that one instance for every `@Test` method, instead of the normal per\-method fresh instance — that's what lets `platform`, `listingRegistry`, and the rest survive from one step to the next as ordinary instance fields, set once in `step01` and read by every step after it. `@TestMethodOrder(MethodOrderer.OrderAnnotation.class)` turns off JUnit's default (deliberately unspecified) method ordering and sorts by the `@Order(n)` value on each method instead, so the eleven steps run in exactly the sequence the old `main()` used — which matters here specifically because later steps depend on state earlier steps built.

The eleven steps mirror the original `main()`'s Examples 1–11 one for one: flat AND query and inventory count, OR\-of\-groups, NOT, role\-based access (posting as seller vs. buyer, then switching roles), the full listing lifecycle (draft → publish → buy → confirm → sold, plus a cancel\-and\-return\-to\-available path), the Observer\-based saved\-search alert (subscribe, publish a match, publish a non\-match, unsubscribe), the Decorator\-based dynamic price quote, the Chain\-of\-Responsibility validation chain (bad price, missing location, duplicate, then a clean pass), the Adapter\-based payment gateways (a successful charge reaching `SOLD`, a failed charge blocking it), the Command pattern's undo (a genuine inverse undo, then a non\-undoable command correctly refusing), and finally the Abstract Factory market bundles (a price the domestic cap rejects but the NRI cap accepts, quoted at each market's own tax rate). Each step is still a real, independently reportable `@Test` method with its own assertions — a failure in step 7 is reported as `SmokeTest#step07_dynamicPricingDecorator`, not as a crash halfway through a giant method — the difference from the granular tests is only that the *fixtures* are shared and the *order* is enforced, not that the assertions are any less specific.

This is intentionally additive, not a replacement for step 19's suite. The granular tests stay the primary line of defense — fast, isolated, order\-independent, and precise about what broke. `SmokeTest` is the one place a regression that only appears under shared state, or only appears when steps run in the platform's natural sequence, has anywhere to surface. Cascading failure is a feature here: if `step04_roleBasedAccess` leaves the buyer in the wrong role, every later step that depends on that role assignment failing loudly is more useful than silently passing.

Verification followed the same offline path as step 19, for the same reason — this cloud sandbox's network allowlist doesn't include Maven Central, so `mvn test` can't resolve JUnit 5 here. The hand\-built JUnit\-5\-API stand\-in from step 19 needed two small extensions to support this file: `@TestInstance`/`TestInstance.Lifecycle`, `@TestMethodOrder`, `@Order`, and a `MethodOrderer.OrderAnnotation` marker type, plus reflection\-runner logic to build one shared instance when `PER_CLASS` is present and to sort `@Test` methods by `@Order` when `OrderAnnotation` is requested. With those in place, the actual `SmokeTest.java` source — not a rewritten stand\-in — compiled and ran cleanly alongside the nine step\-19 test classes: 47 tests total (36 granular \+ 11 smoke steps), all passing, in the correct order, including the expected alert println firing mid\-sequence during step 6. That's a solid proxy but, as with step 19, not a substitute for running `mvn test` for real in an environment with Maven Central access — worth doing once to confirm the real JUnit 5 engine agrees.
