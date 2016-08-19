# Front Matter

TODO

# Minimal executor category

## Executor traits

### Associated future type

    template<class Executor, class T>
    struct executor_future
    {
      // TODO: we can elaborate this in future proposals to allow executor-specific future types
      using type = std::future<T>;
    };
    
    template<class Executor, class T>
    using executor_future_t = typename executor_future<Executor,T>::type;

### Classifying forward progress guarantees of executor operations

    // XXX this section could use a bikeshed

    struct possibly_blocking_execution_tag {};
    struct blocking_execution_tag {};
    struct nonblocking_execution_tag {};
    
    template<class Executor>
    struct executor_operation_forward_progress
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::operation_forward_progress;
    
      public:
        using type = std::experimental::detected_or_t<possibly_blocking_execution_tag, helper, Executor>;
    };

    template<class Executor>
    using executor_operation_forward_progress_t = typename executor_operation_forward_progress<Executor>::type;

## `Executor`

1. The `Executor` requirements form the basis of the executor concept taxonomy;
   every executor satisfies the `Executor` requirements. This set of
   requirements specifies operations for creating execution agents.

2. In Table \ref{executor_requirements}, `f` denotes a `MoveConstructible` function object with zero arguments whose result type is `R`,
   and `x` denotes an object of type `X`.

3. A type `X` satisfies the `Executor` requirements if:
  * `X` satisfies the `CopyConstructible` requirements (17.6.3.1).
  * For any `f` and `x`, at least one of the expressions in Table \ref{executor_requirements} are valid and have the indicated semantics.

Table: (Executor requirements) \label{executor_requirements}


| Expression                       | Return Type              |  Operational semantics                                       | Assertion/note/pre-/post-condition                                                                                              |
|----------------------------------|--------------------------|--------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| `x.spawn_execute(std::move(f))`  | `void`                   |  Creates an execution agent which invokes `f()`              | Effects: blocks the forward progress of the caller until `f` is finished as given by `executor_operation_forward_progress_t<X>` |
| `x.async_execute(std::move(f))`  | `executor_future_t<X,R>` |  Creates an execution agent which invokes `f()`              | Effects: blocks the forward progress of the caller until `f` is finished as given by `executor_operation_forward_progress_t<X>` |
|                                  |                          |  Returns the result of `f()` via the resulting future object |                                                                                                                                 |
 
XXX it's not clear this table can be formatted nicely for a PDF, so we might want to look into an alternate way to specify these requirements

# Bulk (Parallelism TS) executor category

## Bulk executor traits

### Classifying forward progress guarantees of groups of execution agents

    struct sequenced_execution_tag {};
    struct parallel_execution_tag {};
    struct unsequenced_execution_tag {};

    // TODO: we can define this category in a future proposal
    // struct concurrent_execution_tag {};

    template<class Executor>
    struct executor_execution_category
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::execution_category;

      public:
        using type = std::experimental::detected_or_t<unsequenced_execution_tag, helper, Executor>;
    };

    template<class Executor>
    using executor_execution_category_t = typename executor_execution_category<Executor>::type;

### Associated shape type

    template<class Executor>
    struct executor_shape
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::shape_type;
    
      public:
        using type = std::experimental::detected_or_t<size_t, helper, Executor>;

        // exposition only
        static_assert(std::is_integral_v<type>, "shape type must be an integral type");
    };

    template<class Executor>
    using executor_shape_t = typename executor_shape<Executor>::type;

### Associated index type

    template<class Executor>
    struct executor_index
    {
      private:
        // exposition only
        template<class T>
        using helper = typename T::index_type;

      public:
        using type = std::experimental::detected_or_t<executor_shape_t<Executor>, helper, Executor>;

        // exposition only
        static_assert(std::is_integral_v<type>, "index type must be an integral type");
    };

    template<class Executor>
    using executor_index_t = typename executor_index<Executor>::type;

## `BulkExecutor`

1. The `BulkExecutor` requirements form the basis of the bulk executor concept taxonomy.
   This set of requirements specifies operations for creating groups of execution agents in bulk from a single operation.

2. In Table \ref{bulk_executor_requirements},
    * `f` denotes a `CopyConstructible` function object with three arguments,
    * `n` denotes a shape object whose type is `executor_shape_t<X>`.
    * `rf` denotes a `CopyConstructible` function object with one argument whose result type is `R`,
    * `sf` denotes a `CopyConstructible` function object with one argument whose result type is `S`,
    * `i` denotes an object whose type is `executor_index_t<X>`,
    * `r` denotes an object whose type is `R`, 
    * `s` denotes an object whose type is `S`, and
    * `pred` denotes a future object whose result is `pr`.

2. A class `X` satisfies the requirements of a bulk executor if `X` satisfies
   the `Executor` requirements and the expressions of Table
   \ref{bulk_executor_requirements} are valid and have the indicated semantics.

Table: (Bulk executor requirements) \label{bulk_executor_requirements}

| Expression                               | Return Type              |  Operational semantics                                                                                     | Assertion/note/pre-/post-condition                                                                                                                     |
|------------------------------------------|--------------------------|------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `x.bulk_execute(f, n, rf, sf)`           | `R`                      |  Creates a group of execution agents of shape `n` which invoke `f(i, r, s)`                                | Note: blocks the forward progress of the caller until all invocations of `f` are finished.                                                             |
|                                          |                          |  Returns the result of `rf(n)`                                                                             | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            |                                                                                                                                                        |
| `x.bulk_async_execute(f, n, rf, sf)`     | `executor_future_t<X,R>` |  Creates a group of execution agents of shape `n` which invoke `f(i, r, s)`                                | Effects: blocks the forward progress of the caller until all invocations of `f` are finished as required by `executor_operation_forward_progress_t<X>` |
|                                          |                          |  Asynchronously returns the result of `rf(n)` via the resulting future object                              | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            |                                                                                                                                                        |
| `x.bulk_then_execute(f, n, rf, pred, sf)`| `executor_future_t<X,R>` |  Creates a group of execution agents of shape `n` which invoke `f(i, r, pr, s)` after `pred` becomes ready | Effects: blocks the forward progress of the caller until all invocations of `f` are finished as required by `executor_operation_forward_progress_t<X>` |
|                                          |                          |  Asynchronously returns the result of `rf(n)` via the resulting future.                                    | Effects: invokes `rf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            | Effects: invokes `sf(n)` on an unspecified execution agent.                                                                                            |
|                                          |                          |                                                                                                            | If `pred`'s result type is `void`, `pr` is ommitted from `f`'s invocation.                                                                             |
|                                          |                          |                                                                                                            | Post: `pred` is invalid if it is not a shared future.                                                                                                  |

XXX TODO: need to specify how `executor_execution_category_t` describes the forward progress requirements of a group of execution agents wrt each other

XXX it's not clear this table can be formatted nicely for a PDF, so we might want to look into an alternate way to specify these requirements

# (Networking TS) executor category

TODO

# Executor Customization Points

## `execution::spawn_execute()`

    template<class Executor, class Function>
    void spawn_execute(Executor& exec, Function&& f)

TODO: specify semantics

## `execution::async_execute()`

    template<class Executor, class Function>
    executor_future_t<
      Executor,
      result_of_t<decay_t<Function>()>
    >
    async_execute(Executor& exec, Function&& f)

TODO: specify semantics

## `execution::bulk_execute()`

    template<class Executor, class Function1, class Function2, class Function3>
    result_of_t<Function2(executor_shape_t<Executor>)>
    bulk_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                 Function2 result_factory, Function3 shared_factory)

TODO: specify semantics

## `execution::bulk_async_execute()`

    template<class Executor, class Function1, class Function2, class Function3>
    executor_future_t<
      Executor,
      result_of_t<Function2(executor_shape_t<Executor>)>
    >
    bulk_async_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                       Function2 result_factory, Function3 shared_factory)

TODO: specify semantics

## `execution::bulk_then_execute()`

    template<class Executor, class Function1, class Future, class Function2, class Function3>
    executor_future_t<
      Executor,
      result_of_t<Function2(executor_shape_t<Executor>)>
    >
    bulk_then_execute(Executor& exec, Function1 f, executor_shape_t<Executor> shape,
                      Future& predecessor,
                      Function2 result_factory, Function3 shared_factory)

TODO: specify semantics

## Networking TS-specific customization points

TODO

# Execution policy interoperation

## Associated executor

`::executor_type`

TODO

## Execution category

TODO

Describes forward progress guarantees required of groups of execution agents
induced by the execution policy when composed with a control structure. Can
be weaker than the associated executor's guarantee but may not be stronger.

## `.on()`

TODO

# Control structure interoperation

## `std::async()`

    template<class Executor, class Function, class... Args>
    executor_future_t<Executor, result_of_t<decay_t<Function>(decay_t<Args>...)>>
    async(Executor& exec, Function&& f, Args&&... args)

TODO: specify semantics

## `std::future::then()`

TODO: specify semantics

## `std::shared_future::then()`

TODO: specify semantics

## `std::invoke()`

    template<class Executor, class Function, class... Args>
    result_of_t<F&&(Args&&...)>
    invoke(Executor& exec, Function&& f, Args&&... args)

TODO: specify semantics

## `define_task_block()`

TODO

# Thread pool type

TODO

