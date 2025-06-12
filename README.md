## Maintain

This section explains how to maintain the package.

1. We use the following packages to keep the imports clean.

- [import_sorter](https://pub.dev/packages/import_sorter)

Run
```
dart run import_sorter:main
```

2. We are using the following package to index the files from src.

- [index_generator](https://pub.dev/packages/index_generator)

Run
```
dart global run index_generator
```

3. We are using the following package to generate our test mocks.

- [mockito](https://pub.dev/packages/mockito)

Run
```
dart run build_runner build --build-filter="test/**.dart"
```

## Use

This section explains how to use this package in other projects. 

### Setup in the project
To setup this package in another project put a reference to clean_architecture_annotations in the regular dependencies: 
```yaml
dependencies:
  clean_architecture_annotations:
    git:
      url: git@github.com:dgs-games-builder/dart-generators.git
      ref: {{INSERT BRANCH OR COMMIT NUMBER}}
      path: packages/clean_architecture_annotations
```

And put a reference to this package in the dev dependencies:
```yaml
dev-dependencies:
  clean_architecture_generator:
    git:
      url: git@github.com:dgs-games-builder/dart-generators.git
      ref: {{INSERT BRANCH OR COMMIT NUMBER}}
      path: packages/clean_architecture_generator
```

To use the builder we need to make a build.yaml like this:
```yaml
targets:
  $default:
    builders:
      {{INSERT BUILDERS HERE}}
    sources:
      {{INSERT SOURCES HERE}}      
```
What builders and sources you use may vary, refer to example/build.yaml to setup the different builders.
Every builder is set up somewhat like this:
```yaml
clean_architecture_generator|bloc_list_state_builder:
    enabled: true
    generate_for:
      - lib/src/domain/**/use_cases/*.dart
    options:
      srcFolder: true
```
The input path depends on the builder that you're using and what files it should be ran on. srcFolder should be set to true if the directory structure follows:
'lib/src/feature' instead of 'lib/feature'. Otherwise setting it to false explicitly is the best option. 

### Use cases
Most generation is based around use case files. These use cases hold the declaration of a function in a repository. 
Use cases are all set up in a similar way and should be located under '{feature}/use_cases' with a file name like: '{name}_use_case.dart'.

All use case annotations share some parameters:
- **repositoryDefinition**: Required parameter that specifies what repository this use case is tied to to help repository generation. Explained in more detail [here](#repository-generation).
- **protocol**: Is an optional parameter that specifies what protocol is used for the method in the repository. Explained in more detail [here](#repository-generation).
- **blocSettings**: Is an optional parameter that specifies settings for the BLoC generation, explained in more detail [here](#blocsettings).
- **commandOptions**: Is an optional parameter that specifies settings for the command generation, explained in more detail [here](#commandOptions).



The type of use case mainly functions to categorize the use case based on what type of operation it performs. It is also used by the BLoC generator to determine what type of BLoC should be generated.

There are 5 types of use cases:

#### Create
```dart
@CreateUseCase( 
  repositoryDefinition: CollectableRepositoryDefinition, 
  protocol: GrpcCreate.private(),
  blocSettings: BlocSettings(),
)
Future<Either<Failure, Collectable>> createCollectable({
  required String name,
}){}
```

#### Read
```dart
@ReadUseCase(
  repositoryDefinition: CollectableRepositoryDefinition,
  protocol: GrpcRead.private(),
  blocSettings: BlocSettings(
    hasUpdate: true,
    hasClear: true,
  ),
)
Future<Either<Failure, Collectable>> readCollectable({
  required String name,
}){}
```

#### Update
```dart
@UpdateUseCase(
  repositoryDefinition: CollectableRepositoryDefinition,
  protocol: GrpcUpdate.private(),
  blocSettings: BlocSettings(),
)
Future<Either<Failure, Collectable>> updateCollectable({
  required String name,
}){}
```
#### Delete
```dart
@DeleteUseCase(
  repositoryDefinition: CollectableRepositoryDefinition,
  protocol: GrpcDelete.private(),
  blocSettings: BlocSettings(),
)
Future<Either<Failure, Success>> deleteCollectable({
  required String name,
}){}
```

#### List
A list BLoC can either use pagination or infinite scrolling.
This setting determines what kind of information will be stored within the states of the BLoC. For pagination this is a "ListViewModel" and for infinite scrolling this is a list of ListViewModels. 

```dart
@DeleteUseCase(
  repositoryDefinition: CollectableRepositoryDefinition,
  listingType: ListingType.infinite(), // or ListingType.paginate()
  protocol: GrpcDelete.private(),
  blocSettings: BlocSettings(),
)
Future<Either<Failure, CollectableList>> listCollectables(){}
```


### Repository Generation
Repositories within the features are generated from the use cases in the feature. Determine however what repository is tied to what use case and what the repository should be named is difficult without help. That is why another file called a "repository definition" is required to generate the repository. 

Repository definitions should always be located in '{feature}/repository_definitions' with a file name like: '{name}_repository_definition.dart'.

Repository definitions are very simple, every single one looks like this (Collectable is the name of the repository):
```dart
abstract final class CollectableRepositoryDefinition {}
```
The generator then uses this repository definition and all use cases that use this definition in their use case to generate the repository. The repository is generated under '{feature}/repositories' with the file name: '{name}_repository.dart'.

If a protocol is provided in the use case parameters then the corresponding annotation for that protocol is put on top of the method. If at least one method in the repository has a protocol than the repository also gets an annotation that stimulates generation for the infrastructure layer:

```dart
@Repository() 
abstract class CollectableRepository {
  
  @GrpcRead.private()
  Future<Either<Failure, Collectable>> readCollectable({
    required String name,
  }){}
}
```

The protocol is set in the use case by giving an instance of a protocol. Most protocols have either a private or a public constructor that determines what failures need to be accounted for.


### BLoCs
Every generated BLoC has a file that contains the cubit and a file that contains the state. The state is a part file of the cubit. The state file contains 5 different states that are based on the method the BLoC is generated from. This is what a read state file generally looks like:

!!! ReadCollectable is the placeholder use case. File can slightly change depending on type of BLoC.

```dart
part of 'read_collectable_cubit.g.dart';

sealed class ReadCollectableState extends Equatable {
  const ReadCollectableState();

  @override
  List<Object> get props => <Object>[];
}

final class ReadCollectableInitial extends ReadCollectableState {
  const ReadCollectableInitial();
}

final class ReadCollectableInProgress extends ReadCollectableState {
  const ReadCollectableInProgress();
}

final class ReadCollectableSuccess extends ReadCollectableState {
  const ReadCollectableSuccess({
    required this.value,
  });

  final CollectableViewModel value;

  @override
  List<Object> get props => <Object>[value];
}

final class ReadCollectableError extends ReadCollectableState {
  const ReadCollectableError({
    required this.failure,
  });

  final Failure failure;

  @override
  List<Object> get props => <Object>[failure];
}

```

The cubit file contains a cubit class with the execute method of the BLoC. This execute method calls the method in the repository and emits a state based on the result of that call. A read cubit file generally looks like this:

!!! ReadCollectable is the placeholder use case. File changes significantly depending on type of BLoC. [BlocSettings](#blocsettings) can make changes as well.

```dart
// imports
part 'read_collectable_state.g.dart';

@Immutable()
class ReadCollectableCubit extends Cubit<ReadCollectableState> {
  ReadCollectableCubit({
    required Logger logger,
    required CollectableRepository repository,
  })  : _logger = logger,
        _repository = repository,
        super(const ReadCollectableInitial());

  final Logger _logger;
  final CollectableRepository _repository;

  Future<void> execute({
    required String name,
  }) async {
    emit(const ReadCollectableInProgress());

    _logger.debug(<String, dynamic>{
      'method': '$ReadCollectableCubit.execute()',
      'name': name,
    });

    final Either<Failure, Collectable> result = await _repository.ReadCollectable(
      name: name,
    );

    result.fold(
      (Failure failure) {
        _logger.error(
          '$ReadCollectableCubit.execute(): Failed to ReadCollectable',
          error: failure,
        );

        emit(
          ReadCollectableError(
            failure: failure,
          ),
        );
      },
      (Collectable value) {
        emit(
          ReadCollectableSuccess(
            value: CollectableMapper.viewModelFromEntity(value),
          ),
        );
      },
    );
  }
}
```
Depending on the listing type set in a List BLoC the cubit will drastically change. A List BLoC thas uses infinite scrolling will have an extra block of functionality before the repository method call:

```dart
    late final List<CollectableListViewModel> previousValues;

    if (state is ListCollectablesSuccess) {
      final ListCollectablesSuccess successState = state as ListCollectablesSuccess;

      if (successState.values.last.nextCursor.isEmpty) {
        return;
      }

      previousValues = successState.values;
    } else {
      previousValues = <CollectableListViewModel>[];
    }

    emit(
      ListCollectablesInProgress(
        previousValues: previousValues,
      ),
    );
```




#### BlocSettings
BlocSettings specify settings that are used in the generation of the BLoC for the use case.
If generation of the BLoC is wanted, a BlocSettings object should be given.
There are a few settings that modify the generation:

##### hasSuccessCallback
Determines whether a success callback for the cubit should be generated.
This callback will be required to pass through the arguments of the cubit constructor.
The callback will determine what is emitted in the success state of the cubits execute function.

##### hasFailureCallback
Determines whether a failure callback for the cubit should be generated.
This callback will be required to pass through the arguments of the cubit constructor.
The callback will determine what is emitted in the failure state of the cubits execute function.

##### hasClear
!!!!This functionality is only available for the read BLoCs
Determines whether a clear function should be generated alongside the cubit.

A clear function always looks like this (ReadUseCase changes based on your use case:

```dart
void clear() {
  emit(const ReadUseCaseInitial()); 
}
```

##### hasUpdate
!!!This functionality is only available for read and list BLoCs
Determines whether an update function should be generated alongside the cubit.
The update function will have an update callback parameter that will be required to pass through the arguments of the cubit constructor.

In read BLoCs this update function will look like (EntityViewModel and ReadUseCase change based on your use case):
```dart
void update({
  required EntityViewModel value,
}) {
  final ReadUseCaseState currentState = state;
  
  if (currentState is! ReadUseCaseSuccess) {
    return;
  }
  
  emit(
    ReadUseCaseSuccess(
      value: _updateCallback(value, currentState),
    ),
  );
}
```

In list BLoCs it looks slightly different because of the values parameter in the success state:
```dart
void update({
  required EntityViewModel value,
}) {
  final ListUseCaseState currentState = state;
  
  if (currentState is! ListUseCaseSuccess) {
    return;
  }
  
  emit(
    ListUseCaseSuccess(
      values: _updateCallback(value, currentState),
    ),
  );
}
```

### Failures
The builder generates failures for every repository at the path: '{feature}/failure/{repository_name}_failures.g.dart'.

Every failure is fairly simply structured like:
```dart
class ExampleFailure extends Failure {
  const ExampleFailure({
    super.reason,
    super.delegate,
  });
}
```


### Commands
Specifically for the CLI project, the generator also contains functionality to generate the commands and their command use cases.
This can be achieved by using the commandOptions parameter in the use case.

#### CommandOptions
##### shouldGenerateUseCase
Determines whether the command use case should be generated for the command.

##### shouldGenerateCommand
Determines whether the command structure itself should be generated for the command.

##### subCommands
An optional list of subcommands that can be provided. Will be added as subcommands in the command structure.

##### arguments
Arguments of the command should be provided here. Every argument has its own settings that can be set through the ArgumentInfo structure. 

##### failures
An optional list of failures that may be required for the command. This ensures that the failure generator will generate those failures for the command.
