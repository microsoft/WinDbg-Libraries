# Introduction
The [Data Model C++ Object Interfaces](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/data-model-cpp-interfaces) to the data model can be very verbose to implement. While they allow for full manipulation of the data model, they require implementation of a lot of small interfaces to extend the data model (e.g.: an IModelPropertyAccessor implementation for each dynamic fetchable property which is added). In addition to this, the HRESULT based programming model adds a significant amount of boiler plate code in terms of error checking.

In order to alleviate some of this, we have a full C++ helper library for the data model which embraces a full C++ exception and template programming paradigm. Both consumption of the data model and extension of the data model can have considerably more concise code with this library than writing against the COM API.

# Building
The easiest way to use this is through our nuget feed:
* TODO - Add nuget link

For an example, you can start from the sample in our [WinDbg-Samples repo](https://github.com/Microsoft/WinDbg-Samples):
* TODO - Add full sample link

# How to use
There are two important namespaces in the helper library:

* `Debugger::DataModel::ClientEx` - helpers for consumption of the data model
* `Debugger::DataModel::ProviderEx `- helpers for extension of the data model

In addition, any client which uses this library must provide implementations of the following methods for the library to function:

```cpp
namespace Debugger::DataModel::ClientEx
{
    IDataModelManager *GetManager();
    IDebugHost *GetHost();
}
```

## Consuming the Data Model - `Debugger::DataModel::ClientEx`

The primary class which represents an object in the data model is `Object`. It is somewhat a drop-in replacement for where one might use ComPtr<IModelObject> in code written against the raw COM API. There are a few important notes about `Object`:

### Creation of Objects
Many types are automatically convertible to Object. You can simply construct or assign to an Object with a native type:
```cpp
Object intObj = 42;
Object doubleObj = 3.5;
Object boolObj = true;
```

C++ strings are also convertible to an object:
```cpp
std::wstring myString = L"Hello World";
Object stringObj = myString;
```

Lambda methods and free floating C++ functions are convertible to an object:

 ```cpp
//
// Create a model method (myAddFunction) which adds two integers and returns the result as integer. Everything is ``type inferred``. The code
// generated underneath the below will automatically return E_INVALIDARG if two arguments are not passed to the dynamic call. It will automatically
// return E_INVALIDARG if the two arguments which are passed are not coercible to int. It will automatically box the returned integer back into
// an object.
//
Object myAddFunction = [](const Object& /*contextObject*/, int a, int b) { return a + b; };

int FreeFloatingAdder(const Object& /*contextObject*/, int a, int b)
{
  return a + b;
}

//
// Create a model method (myFreeFloatingAdder) similar to the above. These two are indistinguishable from the data model perspective.
//
Object myFreeFloatingAdder = FreeFloatingAdder;
 ```

There are also a set of factory methods to construct objects:

 ```cpp
// Get the root namespace (where you'll find "Debugger", etc...)
static Object RootNamespace();

// Get a boxed representation of the current context
static Object CurrentContext();

// Get the session that is the current UI state of the debugger.
static Object CurrentSession();

// Get the process that is the current UI state of the debugger.
static Object CurrentProcess();

// Get the thread that is the current UI state of the debugger.
static Object CurrentThread();

// Get the session associated with a given object. If no session is associated, this throws.
static Object SessionOf(_In_ const Object& obj);

// Get the process associated with a given object. If no process is associated, this throws.
static Object ProcessOf(_In_ const Object& obj);

// Get the thread associated with a given object. If no thread is associated, this throws.
static Object ThreadOf(_In_ const Object& obj);

// Create a new synthetic object with an optional host context assigned and an optional set of properties initialized.
// Initializer arguments must follow the form [name, value] where name is a string ([const] wchar_t*, [const] std::wstring[&]) and value is anything that can be boxed.
template<typename... TArgs> static Object Create(_In_ const HostContext& hostContext, _In_ TArgs&&... initializers);

// Create a new pointer object with the given type
static Object CreatePointer(_In_ const Type& pointerType, _In_ ULONG64 ptrValue);

// Create an object from a symbol
static Object FromSymbol(_In_ IDebugHostSymbol *);

// Get a global symbol
template<typename TStr1, typename TStr2> static Object FromGlobalSymbol(_In_ const HostContext& symbolContext, _In_ TStr1&& moduleName, _In_ TStr2&& symbolName);
 ```

### Use Of Objects 

#### Unboxing
There are a number of ways to use Objects once they are acquired. Unboxing is a matter of casting or calling the As method. If the object cannot naturally convert to the given type, a C++ exception will be thrown.

 ```cpp
int intValue = (int)intObj;
double doubleValue = (double)doubleObj;
bool boolValue = boolObj.As<bool>();
 ```

#### Keys and Fields
Keys (synthetic "properties") and fields (native "properties") on an object can be accessed thorugh the Keys() and Fields() methods. Each of these methods returns things which are inherently C++ reference style objects:

 ```cpp
// Get a reference to a native field called m_intVal:
Object myStruct = GetMyStruct();
auto&& fieldRef = myStruct.Fields()[L"m_intVal"];

// Get the int value stored in the field
int intVal = (int)fieldRef;

// Set the int value stored in the field
fieldRef = 42;

// Get a reference to a key called ProjectedValue:
auto&& keyRef = myStruct.Keys()[L"ProjectedValue"];

// Get the int value stored in the key
intVal = (int)keyRef;

// Set the int value stored in the key
keyRef = 99;
 ```

There is some overhead in returning the intermediate objects necessary to have reference and assignment semantics like this function. If you only want the values, the `FieldValue` and `KeyValue` methods bypass this:

 ```cpp
Object myStruct = GetMyStruct();

// Get the integer field value.
Object intFieldValue = myStruct.FieldValue(L"m_intVal");
int intVal = (int)intFieldValue;

Object keyValue = myStruct.KeyValue(L"ProjectedValue");
intVal = (int)keyValue;
 ```

Fields and keys can also be enumerated through standard C++ means:

 ```cpp
Object myObject = GetMyObject(); // get some object with keys and fields
for (auto&& kvm : myObject.Keys())
{
    // kvm is a tuple of <name, value, metadata>
    const std::wstring& keyName = std::get<0>(kvm);
    Object keyValue = std::get<1>(kvm);
    Metadata keyMetadata = std::get<2>(kvm);
    
    // Do something with the key name, value, and metadata
}

for (auto&& kv : myObject.Fields())
{
    // kv is a pair of <name, value>
    const std::wstring& fieldName = kv.first;
    Object fieldValue = kv.second;
    
    // Do something with the field name and value
}
 ```

#### Iterating Objects
Any iterable object can be iterated through standard C++ means:

 ```cpp
Object myVector = GetMyVector(); // get a std::vector<int> {1, 2, 3, 4, 5} in the target process

int expected = 0;
for (auto&& vectorItem : myVector)
{
    int value = (int)vectorItem;
    VERIFY_IS_TRUE(expected == value);
}
 ```

#### Indexing Objects
Any indexable object can be indexed through the standard C++ index operator []. Data model objects can be indexed in multiple dimensions and with varying types. An out of bounds indexing will result in an exception being thrown.

 ```cpp
Object myVector = GetMyVector(); // get a std::vector<int> {1, 2, 3, 4, 5} in the target process

for (int i = 0; i < 5; ++i)
{
    Object vectorItem = myVector[i];
    int value = (int)vectorItem;
    VERIFY_IS_TRUE(expected == value);
}
 ```

#### Calling Objects
There are two ways to call an object, through the ``Call`` method and through the ``CallMethod`` method. The explicit ``Call`` method requires passing the context object (this pointer) manually. ``CallMethod`` does this automatically. A basic example of creating and calling methods through the data model:

 ```cpp
// Create a method that adds integers:
Object myAdder = [](_In_ const Object& /*contextObject*/, int a, int b) { return a + b; }

// Call the method explicitly through the data model:
Object result = myAdder.Call(Object() /* no context object -- free floating */, 5, 7);

// Unbox the result and verify:
int resultInt = (int)result;
VERIFY_IS_TRUE(resultInt == 12);
 ```

A more complex example of using a LINQ query from C++ via the library:

 ```cpp
Object myVector = GetMyVector(); // get a std::vector<int> {1, 2, 3, 4, 5} in the target process

Object queryResult = myVector.CallMethod(L"Where", [](_In_ const Object& /*contextObj*/, _In_ int val) { return val < 3; })
               .CallMethod(L"Select", [](_In_ const Object& /*contextObj*/, _In_ int val) { return val * 10; });

for (auto&& queryItem : queryResult)
{
  //
  // You will see the queryItem sequentially produce 0, 10, and 20 (the ``Where`` clause having filtered anything that doesn't meet "val < 3"
  // and the ``Select`` clause having multiplied each item by 10.
  //
}
 ```

## Extending the Data Model (``Debugger::DataModel::ProviderEx``)

The basic idea of the helper library is that you implement a C++ class for every data model you wish to provide. Each of these classes implements property getters, setters, and methods as appropriate. A single instance of the class is instantiated. That instance acts as either the binding for the extensibility point or as a "type factory" for some synthetic type.

### Basic Extensions: The ``ExtensionModel`` Class
A data model designed to extend something else (whether that extension is for a native type signature or some debugger concept like process) is a class which derives from ``ExtensionModel``. The ``ExtensionModel`` constructor is passed a set of registration records which tell the library how the class binds to the data model. These are similar in spirit to the registration records passed to JavaScript extensions in the ``initializeScript`` method.

The current supported records are ``TypeSignatureRegistration``, ``TypeSignatureExtension``, ``NamedModelParent``, and ``NamedModelRegistration``.

An example of extending a native type ``_MYSTRUCT``:

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
  }
};

//
// Bind the extension. The instantiation here will associate everything with the data model. The instant this class destructs, the extension is detached from the data model.
// That does not mean that no COM interfaces are alive! Such interfaces will effectively act as weak pointers with a broken link (they will fail).
//
// This is an exception model. The constructor here may throw for a variety of reasons (inability to register, etc...)
//
MyStructExtension xtn;
 ```

#### Adding Properties
Properties (keys) can be added to the data model via the ``AddProperty`` or ``AddReadOnlyProperty`` methods. These methods bind a class method as the property getter/setter. What types your property returns or accepts is determined via type analysis of your methods. Object indicates "any object". A concrete type must match or an exception is thrown.

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
    AddReadOnlyProperty(L"NewProperty", this, &MyStructExtension::Get_NewProperty);
  }

  int Get_NewProperty(_In_ const Object& myStruct)
  {
    //
    // Let's just return the value of some native field "m_intVal" plus one.
    //
    return (int)myStruct.FieldValue(L"m_intVal") + 1;
  }
};
 ```

#### Adding Callable Methods
Instance methods can be added to the data model via the ``AddMethod`` method. The types and number of your input arguments is inferred from the signature of the method provided to ``AddMethod``. If the dynamic arguments provided to the call can be coerced to the signature types, the method is called; otherwise, an E_INVALIDARG is returned. An argument which is Object (or const Object&) can take any type.

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
    AddMethod(L"AddValues", this, &MyStructExtension::AddValues);
  }

  int AddValues(const Object& /*contextObject*/, int a, int b)
  {
    return a + b;
  }
};
 ```

Methods can take optional arguments via having any argument slot be std::optional<TOpt>. Only std::optional<TOpt> and variable arguments (see below) can follow the first std::optional<TOpt> in a method signature. The advantage of optional arguments over variable arguments is that the client library will do type checking on TOpt for you.

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
    AddMethod(L"SumValues", this, &MyStructExtension::SumValues);
  }

  //
  // The method can be called .CallMethod(L"Sum", a) or .CallMethod(L"Sum", a, b).
  // In either case, the 'a' and 'b' values passed in must be convertible to int
  // or the library will throw an invalid_argument.
  //
  int SumValues(const Object & /*contextObject*/, int startVal, std::optional<int> addend)
  {
    return startVal + addend.value_or(0);
  }
};
 ```

Methods can also take variable arguments via having the last two arguments be "size_t, Object *". The method may optionally have static arguments (which must be present) before the variable argument marker. If a method takes variable arguments, it is responsible for doing any type validation it wants.

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
    AddMethod(L"SumValues", this, &MyStructExtension::SumValues);
    AddMethod(L"OtherSumValues", this, &MyStructExtension::OtherSumValues);
  }

  // This method will accept zero or more arguments of any type:
  int SumValues(const Object& /*contextObject*/, size_t numVarArgs, Object *pVarArgs)
  {
    int sum = 0;
    for (size_t i = 0; i < numVarArgs; ++i)
    {
      sum += (int)(pVarArgs[i]);
    }
    return sum;
  }

  // This method will accept something coercible to int followed by zero or more variable arguments.
  // E_INVALIDARG will be returned from the IModelMethod::Call implementation if less than one
  // argument is passed.
  int OtherSumValues(const Object& /*contextObject*/, int first, size_t numVarArgs, Object *pVarArgs)
  {
    return SumValues(numVarArgs, pVarArgs) + first;
  }
};
 ```

Methods can also combine optional and variable arguments.

#### Making Objects Iterable
Making a model iterable is done through binding a generator method on the class. Such method returns a type which is C++ iterable (.begin() / .end()) and the items returned out of that iterator are boxed and returned. Experimental support for co-routines and generator objects makes this significantly easier. Enabling such requires:

* Using UCRT / the latest STL
* Adding ``/await`` to the compiler switches (``USER_C_FLAGS = $(USER_C_FLAGS) /await``)
* Including ``<experimental\generator>``

Consider that we have a C++ type called "Simple1DArray" which has two fields: m_size and m_pValues. m_size indicates the number of elements in the array, and m_pValues is a pointer to their values. The following might make that object iterable as far as the data model is concerned:

 ```cpp
class Simple1DArrayExtension : public ExtensionModel
{
public:
  Simple1DArrayExtension() : ExtensionModel(TypeSignatureRegistration(L"Simple1DArray"))
  {
    AddGeneratorFunction(this, &Simple1DArrayExtension::GetIterator);
  }

  std::experimental::generator<int> GetIterator(_In_ const Object& arrayObject)
  {
    auto&& fields = arrayObject.Fields();
    ULONG64 size = (ULONG64)fields[L"m_size"];
    Object ptr = fields[L"m_pValues"];

    for (ULONG64 i = 0; i < size; ++i)
    {
      co_yield((int)ptr.Dereference());
      ++ptr;
    }
  }
};
 ```

#### Making Objects Indexable
There is a key association between an iterator and an indexer. Generally, if an object is indexable, it is also iterable. The iterator must define a default index and return that index for every element produced. The indexer must be able to get back to each element via the index returned from the iterator. Because of the close association between iterator and indexer, an object is made indexable via binding a generator function as well as a ``getAt`` / ``setAt`` pair.

The iterator returns a special ``IndexedValue`` type to associate indicies with the return value.

 ```cpp
class Simple1DArrayExtension : public ExtensionModel
{
public:
  typedef IndexedValue<int, ULONG64> TIndexedValue;

  Simple1DArrayExtension() : ExtensionModel(TypeSignatureRegistration(L"Simple1DArray"))
  {
    AddIndexableGeneratorFunction(this, &Simple1DArrayExtension::GetIterator, &Simple1DArayExtension::GetAt, &SimpleArray1DExtension::SetAt);
  }

  std::experimental::generator<TIndexedValue> GetIterator(_In_ const Object& arrayObject)
  {
    ULONG64 idx = 0;
    auto&& fields = arrayObject.Fields();

    ULONG64 size = (ULONG64)fields[L"m_size"];
    Object ptr = fields[L"m_pValues"];

    for (ULONG i = 0; i < size; ++i)
    {
      co_yield(TIndexedValue((int)ptr.Dereference(), idx));
      ++ptr;
    }
  }

  int GetAt(_In_ const Object& arrayObject, _In_ ULONG64 idx)
  {
    auto&& fields = arrayObject.Fields();

    ULONG64 size = (ULONG64)fields[L"m_size"];
    Object ptr = fields[L"m_pValues"];

    if (idx >= size)
    {
      throw std::range_error("Array index out of bounds");
    }

    ptr += idx;
    return (int)ptr.Dereference();
  }

  void SetAt(_In_ const Object& arrayObject, _In_ int value, _In_ ULONG64 idx)
  {
    auto&& fields = arrayObject.Fields();
    ULONG64 size = (ULONG64)fields[L"m_size"];
    Object ptr = fields[L"m_pValues"];

    if (idx >= size)
    {
      throw std::range_error("Array index out of bounds");
    }

    ptr += idx;
    ptr.Dereference() = value;
  }
};
 ```

#### Making Objects Have a Display String Conversion
Making an object have a string conversion is a matter of binding a string conversion function on the class and returning a string. As an example:

 ```cpp
class MyStructExtension : public ExtensionModel
{
public:
  MyStructExtension() : ExtensionModel(TypeSignatureRegistration(L"_MYSTRUCT"))
  {
    AddStringDisplayableFunction(this, &MyStructExtension::GetDisplayString);
  }

  std::wstring GetDisplayString(_In_ const Object& myStructObj, _In_ const Metadata& metadata)
  {
    return std::wstring(L"Hello World");
  }
};
 ```

### Type Factories: The ``TypedInstanceModel`` Template Class
The data model is frequently a projection of data stored somewhere else. It can be incredibly useful to have a data model class model or represent some native data structure. The ``TypedInstanceModel<T>`` template is designed to do exactly this -- provide a means of representing instances of a native type in the data model.

``TypedInstanceModel`` is similar to ``ExtensionModel`` in that you derive a class from the ``TypedInstanceModel`` class and bind properties, methods, iterators, indexers, etc... There are a few key differences:

* Since ``TypedInstanceModel<T>`` is designed to be a type factory, it does *NOT* take registration records like ``ExtensionModel`` does.
* You create object instances through the ``CreateInstance`` method on ``TypedInstanceModel<T>``. A singleton instance of the class is the type factory.

In addition, properties, methods, and iterators can have two binding forms:

* They can be function bound. This means that to fetch the property, call the method, get the iterator, etc..., a function on the class is called.
* They can be direct bound. This means that the property/method/iterator is directly bound to something on the native type T of the ``TypedInstanceModel<T>``.

A type factory can hold its data in three ways:

* By value: declare a derivative of ``TypedInstanceModel<T>``.
* By unique pointer: declare a derivative of ``TypedInstanceModel<std::unique_ptr<T>>``
* By shared pointer: declare a derivative of ``TypedInstanceModel<std::shared_ptr<T>``

Regardless of how the data is stored in the above examples, the factory is always for ``T`` and never for ``std::unique_ptr<T>`` or ``std::shared_ptr<T>``.

#### Creating Instances, Type Checking, and Getting Back Native Data
Once a type factory is created for a given type ``T`` (by having a singleton derived ``TypedInstanceModel<T>`` class instantiated), there are a set of conversions between native data and instance objects:

 ```cpp
// Define a native type that we wish to project into the data model
struct MyNativeType
{
  std::wstring Name;
  int Id;
  int WriteableValue;
};

//
// Imagine we defined MyNativeTypeFactory : public TypedInstanceModel<MyNativeType> (omitted for brevity -- see below)
//
MyNativeTypeFactory typeFactory;

// Create an instance:
Object instance = typeFactory.CreateInstance(MyNativeType { L"Hello World", 42, 37 });

// Check whether the instance is an instance of the factory
bool isFactoryInstance = typeFactory.IsObjectInstance(instance);  // result here is true; passing some other object would result in false

// Get back to the underlying native data (this throws for objects which aren't instances of the factory)
MyNativeType& storedData = typeFactory.GetStoredInstance(instance);
 ```

#### Participating in Boxing, Unboxing, and Other Composability 
If you have a type factory defined for some type ``T``, you can provide custom boxing and unboxing support for instances of ``T``. Doing this also adds composability with binding operations. As an example, if you have some type, ``MyNativeType`` and you provide boxing and unboxing, you can direct bind a field of ``MyNativeType`` to a property or string conversion.

Providing custom boxing and unboxing operations requires providing a template specialization of ``BoxObject<T>`` within the ``Debugger::DataModel::ClientEx::Boxing`` namespace:

 ```cpp
//
// Imagine we defined MyNativeTypeFactory : public TypedInstanceModel<MyNativeType> (omitted for brevity -- see below)
//
MyNativeTypeFactory *g_pTypeFactory = new MyNativeTypeFactory;

namespace Debugger::DataModel::ClientEx::Boxing{
  template<>
  struct BoxObject<MyNativeType>
  {
    static Object Box(_In_ const MyNativeType& myNativeType)
    {
      if (g_pTypeFactory == nullptr) { /* throw something */ }
      return g_pTypeFactory->CreateInstance(myNativeType);
    }

    static MyNativeType Unbox(_In_ const Object& src)
    {
      if (g_pTypeFactory == nullptr || !g_pTypeFactory->IsObjectInstance(src)) { /* throw something */ }
      return g_pTypeFactory->GetStoredInstance(src);
    }
  };
} // Debugger::DataModel::ClientEx::Details

// Example usages:
MyNativeType myNativeInst;
Object myInstanceObj = myNativeInst;              // This will automatically make use of ObjxerBoxer<MyNativeType>
MyNativeType returnedNativeInst = (MyNativeType)myInstanceObj; // This will automatically make use of ObjectUnboxer<MyNativeType>

 ```

One could also take a definition across DLL boundaries by having the ``BoxObject<T>::Box`` and ``BoxObject<T>::Unbox`` implementations call export functions.

#### Direct Binding of Properties
To directly bind properties to fields on the native type, either the ``BindProperty`` or ``BindReadOnlyProperty`` methods are called and passed a pointer-to-data-member of ``T``.

 ```cpp
// Define a native type that we wish to project into the data model
struct MyNativeType
{
  std::wstring Name;
  int Id;
  int WriteableValue;
};

// Declare a type factory for the type
class MyNativeTypeFactory : public TypedInstanceModel<MyNativeType>
{
public:
  MyNativeTypeFactory()
  {
    BindReadOnlyProperty(L"Name", &MyNativeType::Name);
    BindReadOnlyProperty(L"Id", &MyNativeType::Id);
    BindProperty(L"WriteableValue", &MyNativeType::WriteableValue);
  }
};

// Create the type factory and make an instance
MyNativeTypeFactory factory;
Object instance = factory.CreateInstance(MyNativeType { L"Foo", 42, 37 });

// There are "Name/Id" read-only properties on instance and a "WriteableValue" property.
 ```

#### Direct Binding of Iterators and Indexers
If the native type being represented by the ``TypedInstanceModel<T>`` has a standard C++ iterator protocol (begin / end), the iterator can be directly bound to the data model. Further, if the iterator returned from that protocol is random access, an indexer will automatically be bound:

 ```cpp
// Define a factory over std::vector<int>
typedef std::vector<int> IntVector;
class MyVectorFactory : public TypedInstanceModel<IntVector>
{
public:
  MyVectorFactory()
  {
    BindIterator();
  }
};

// Create the factory and an instance:
MyVectorFactory vectorFactory;
Object vecObj = vectorFactory.CreateInstance(IntVector { 1, 2, 3, 4, 5 });

// The object can be iterated through the data model.
for (auto&& vecItem : vecObj)
{
  // vecItem is an object. You can convert with casts, etc...
}

// Directly index since std::vector's iterators are random access
int val2 = (int)vecObj[2]; // would be '3'
 ```

#### Direct Binding of Methods
If the native type being represented by the ``TypedInstanceModel<T>`` has instance methods on it, those instance methods can be directly bound to data model methods.

 ```cpp
// Create a type factory over std::vector<int>
typedef std::vector<int> IntVector;
class MyVectorFactory : public TypedInstanceModel<IntVector>
{
public:
  MyVectorFactory()
  {
    // Bind the iterator
    BindIterator();

    // Bind an overload (the const T& variant) of push_back to the data model
    void (IntVector::*pfn)(_In_ const int&) = &IntVector::push_back;
    BindMethod(L"PushBack", pfn);
  }
};

// Create the factory and an instance:
MyVectorFactory vectorFactory;
Object vecObj = vectorFactory.CreateInstance(IntVector { 1, 2, 3, 4, 5 });

// Call the bound method
vecObj.CallMethod(L"PushBack", 6);

// The object can be iterated through the data model.
for (auto&& vecItem : vecObj)
{
  // vecItem is an object. You can convert with casts, etc...
  // 1, 2, 3, 4, 5, 6 will be iterated because of the PushBack call
}
 ```

#### Direct Binding of String Conversion
Display string conversion can be directly bound to a field of the native type and will use the string conversion for whatever the boxing of that type would be:

 ```cpp
// Define a native type that we wish to project into the data model
struct MyNativeType
{
  std::wstring Name;
  int Id;
  int WriteableValue;
};

// Declare a type factory for the type
class MyNativeTypeFactory : public TypedInstanceModel<MyNativeType>
{
public:
  MyNativeTypeFactory()
  {
    BindStringConversion(&MyNativeType::Id);
  }
};

// Create the type factory and make an instance
MyNativeTypeFactory factory;
Object instance = factory.CreateInstance(MyNativeType { L"Foo", 42, 37 });

// Get the string conversion:
std::wstring strConv = instance.ToDisplayString();                // strConv would be "42"
std::wstring strConv = instance.ToDisplayString(Metadata(L"PreferredRadix", 16)); // strConv would be "0x2a"
 ```

#### Function Binding

The function binding style of ``TypedInstanceModel<T> ``is nearly identical to the descriptions of methods in ``ExtensionModel`` with one notable exception:

* All ``ExtensionModel`` function bindings start with ``(const Object& instanceObject, ...)``
* All ``TypedInstanceModel<T>`` function bindings start with ``(const Object& instanceObject, [const] T& instanceData, ...)``
