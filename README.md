# Quick guide of efecticve Bloc test coverag

One of the great advantages of using bloc as a state manager is its ease of testing its components due to the way it works.

However, the benefits of an application do not arise independently of written tests. Many people don't know what to test, and end up testing unnecessary things. The biggest reason for this is because many developers don't know how to write tests..

*So how i should write test?*  
That's what we'll talk about in this post.

<ha-alert alert-type="warning">
  This is not a simple topic and i will asume you have already a notion in flutter and bloc.

---

## Bloc test:

In a perfect world, the ideal test will test all of your lines. First, in a declaration of a bloc we will have something like this:

```dart
class UserBloc extends Bloc<UserEvent, UserState> {
    UserBloc() super(null) {...}
}
```
    
<ha-alert alert-type="info">
    Of course, instead of null you could have a UserState.initial() in your super if you are working whit freezed, for exemple. But the test will be the same, so lets keep it simple. 

<br/><br/>

### Lets start or bloc test file with the normal approach of test scructure:

```dart
void main() {
    group('User bloc testing', () {
        late UserBloc bloc;
        late IAutenticationRepositoryInterface repository;
        late UserDataModel userMockedData;

        setUp(() {
            // This mock can be from any package, or even a self-made class.
            repository = MockedAutenticationRepository();

            // The block to be tested
            bloc = UserBloc(repository);

            // A class with random data just to be returned by the repository
            userMockedData = UserDataModel( ... ); 
        });

        // ==> All tests below will be writes here
    });
}
``` 
<br/><br/> 

### Test the good path

Okay, the initial state is null. How can we garant that no one will change this guy
?  With ```bloc_test``` package, we will do something like this:

```dart
blocTest<UserBloc, UserState>(
    'Initial state must be null',
    // This is a instance of my bloc that i declared in the test.
    build: () => bloc, 
    // Here we are verifing if the initial state is null.
    verify: (bloc) => expect(bloc.state, null), 
);
```
<br/><br/>
Now lets try calling a event and testing it and see if it will pass in the expected states:

```dart
blocTest<UserBloc, UserState>(
    'Must get the users data',
    build: () => bloc,
    // In setUp we put the answer that our mocked repository has to give when the function that we are testing is called.
    setUp: () {
        // Asume the [ UserEvent.getData() ] calls this repository.getDataFromDatabase()
        // that efecticve get the data.
        when(() => repository.getDataFromDatabase()).thenAnswer((_) async => userMockedData);
    },
    // Here, we can set the event that will be emit
    act: (bloc) => bloc.add(const UserEvent.getData()),
    expect: () => [
         // I want it to pass through this state.
        UserState.isLoading(),

        // Here all went well and i espect a user state that has data.
        // The returned data has to be the mocked data.
        UserState.withDataFromDB(userData: userMockedData), 
    ],
);
```

Cool, we just see how to emit a state, make our fake repository return the data that we want (in test case, the mocked response data).

We also saw how we can pass the states that we expect the emitted event to pass.

<br/><br/> 

### I want to test if my function is caching, how?

Okay, lets see a more realistic example.

Using the same structure above, but this time we are gonna call the function two times and expect the **first** result to be from the database (mocked database) and the second call beeing from the cashed data, that has his own state, with the same atributes of the ```UserState.withDataFromDB()```, but cached from the previous request.

```dart
blocTest<UserBloc, UserState>(
    'Must get the users data from database first time and from cache thereafter',
    build: () => bloc,
    setUp: () { 
        when(() => repository.getDataFromDatabase()).thenAnswer((_) async => userMockedData);
    },
    act: (bloc) => bloc.add(const UserEvent.getData()), 
    expect: () => [ 
        UserState.isLoading(),

        // We will call two times the [ repository.getDataFromDatabase() ].

        // First, it has to pass through the state that it has data from the database.
        UserState.withDataFromDB(userData: userMockedData),

        // Then, it has to pass through the state that it has data from the cached files.
        UserState.withDataFromCache(userData: userMockedData),
    ],
);
```

<br/><br/> 

### Test the bad path, with erro. 
Now lets see if the bad scenery, when an error occurs, is happening as expected.

```dart
blocTest<UserBloc, UserState>(
    'Must pass through the error state with the correct error message return',
    build: () => bloc, 
    setUp: () {
        // Asume the [ UserEvent.getData() ] calls this repository.getDataFromDatabase()
        // that efecticve get the data.
        when(() => repository.getDataFromDatabase()).thenThrow(NoInternetError());
    }, 
    act: (bloc) => bloc.add(const UserEvent.getData()), 
    expect: () => [ 
        UserState.isLoading(),
        
        // Lets assume that when this type of error hapends, the repository throws a custom error that contains a message.
        //
        // Here i am seeing if the event [ UserEvent.getData() ] is using the messaging that has beeing passed by the repository.
        UserState.hasError(message: 'Connect your device to internet'),
    ],
);
```

---
