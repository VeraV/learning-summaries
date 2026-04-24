Big picture                                                                                                         
                                                                                                                      
  The error-handling spec (docs/error-handling-spec.md) defines three shared pieces on the client; one of them is the 
  axios response interceptor. Its job, per the spec:                                                                  
                                                                                                                      
  ▎ On 401 response:                                                                                                  
  ▎ 1. Clears localStorage (token, userId, email, name)                                                               
  ▎ 2. Redirects to /login (with loop prevention — skips if already on /login)                                        
  ▎ 3. All other errors bubble to the calling page.                                                                   
                                                                                                                      
  axios.spec.ts is a unit test that pins exactly those four rules down. Nothing more, nothing less — the request      
  interceptor is outside the error-handling spec, so this file doesn't touch it.                                      
                                                                                                                      
  What each test maps to in the spec                                                                                  
   
  ┌─────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────┐   
  │                        Spec line                        │                        Test                         │
  ├─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
  │ "Clears localStorage (token, userId, email, name)" +    │ Test 1: clears auth data … and redirects to /login  │
  │ "Redirects to /login"                                   │ on 401                                              │
  ├─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤   
  │ "loop prevention — skips if already on /login"          │ Test 2: clears storage but does not redirect when   │
  │                                                         │ already on /login                                   │   
  ├─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
  │ "All other errors bubble" (non-401 with response)       │ Test 3: bubbles non-401 errors without touching     │   
  │                                                         │ storage or location                                 │   
  ├─────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────┤
  │ "All other errors bubble" (network error, no response)  │ Test 4: bubbles network errors … without touching   │   
  │                                                         │ storage or location                                 │   
  └─────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────┘
                                                                                                                      
  File structure, top to bottom                                                                                       
   
  1. Reach into the registered handler (axios.spec.ts:3-10)                                                           
  const responseHandler = (
    api.interceptors.response as unknown as { handlers: ResponseHandler[] }                                           
  ).handlers[0];                                                           
  When axios.ts is imported, it calls api.interceptors.response.use(success, error). Axios stores those in an internal
   handlers[] array. Rather than making a real HTTP call and mocking the transport, we grab the registered rejected   
  function directly and invoke it with a fabricated error. That is what "unit test the interceptor" means here — we   
  isolate just the function the spec describes.                                                                    
                                                                                                                      
  2. Patch jsdom's location.href setter (axios.spec.ts:12-36)
  const implSymbol = Object.getOwnPropertySymbols(window.location).find(...)                                          
  const locationImplProto = Object.getPrototypeOf(window.location[implSymbol])
  // replace href setter with jest.fn(), restore in afterAll                                                          
  This is the only exotic piece. jsdom (which Jest uses as its fake browser) treats window.location and location.href 
  as non-configurable by spec — you can't delete, replace, or jest.spyOn them. But internally jsdom stores the real   
  implementation on an object attached via Symbol(impl), and that prototype's href setter is configurable. So we reach
   through the symbol, swap href's setter for jest.fn(), and restore it in afterAll. That's how we get to assert      
  hrefSetter.toHaveBeenCalledWith("/login") without triggering jsdom's "Not implemented: navigation" error.           
                                                                                                           
  3. Reset between tests (axios.spec.ts:38-41)
  localStorage.clear() + hrefSetter.mockClear() so each test starts from a known state.                               
                                                                                                                      
  4. The four tests (axios.spec.ts:43-92) — each follows the same three-beat shape:                                   
  - Arrange: seed localStorage, set the path via window.history.pushState(...) (this works even in jsdom), build a    
  fake AxiosError.                                                                                                    
  - Act: await responseHandler.rejected(error) and assert it re-rejects the same error (spec: "errors bubble").
  - Assert: check localStorage was / wasn't cleared, and that hrefSetter was / wasn't called with /login.


  Deeper dive into the tricky parts of axios.spec.ts                                                                  

  1. What is ResponseHandler?                                                                                         
   
  type ResponseHandler = {                                                                                            
    fulfilled: (value: AxiosResponse) => AxiosResponse | Promise<AxiosResponse>;
    rejected: (error: AxiosError) => unknown;                                                                         
  };
                                                                                                                      
  This is a custom TypeScript type — a local description of the shape we expect to find inside axios's internal       
  handlers[] array. It's not imported from axios; axios doesn't export this type.
                                                                                                                      
  When axios.ts calls api.interceptors.response.use(success, error), axios stores those two callbacks together as a   
  single entry. That entry is an object with:
  - fulfilled — the success callback (takes a response, returns it possibly transformed, or a promise of it)          
  - rejected — the error callback (takes an AxiosError, can do anything — our code returns Promise.reject(error),     
  hence unknown)                                                                                                 
                                                                                                                      
  We wrote this type so that once we dig into the private handlers[0], TypeScript knows what shape we're working with
  — otherwise every access would be any and you'd lose all type safety inside the tests.                              
                  
  Think of it as: "Here's my understanding of axios's internal data structure; please check my code against this."    
                  
  2. Why the double as unknown as?                                                                                    
                  
  api.interceptors.response as unknown as { handlers: ResponseHandler[] }
                                                                                                                      
  This is TypeScript's escape hatch for incompatible type assertions.                                                 
                                                                                                                      
  In TypeScript you can cast A as B directly only if A and B overlap — one must be assignable to the other.           
  api.interceptors.response is typed (by axios) as AxiosInterceptorManager<AxiosResponse>, which publicly exposes just
   three methods: use, eject, clear. There's no handlers property in the public type.                                 
                  
  If you tried to write it in one step:                                                                               
  api.interceptors.response as { handlers: ResponseHandler[] }  // ERROR
  TypeScript would complain: "the two types don't overlap — this cast might be wrong." It's protecting you from       
  accidental bad casts.                                                                                               
                                                                                                                      
  The as unknown step is a reset button. unknown is compatible with everything (it's the "I don't know what this is"  
  type). So the pattern is:                                                                                           
  1. x as unknown — throw away all type information
  2. as NewType — now assert it's whatever you want                                                                   
                                                   
  In short: as unknown as X is how you tell TypeScript "I know better than you do — this value really is shaped like  
  X, trust me." It's essentially a two-step force-cast.                                                               
                                                                                                                      
  We need it here because handlers is private axios internals and the public type intentionally hides it.             
                  
  3. What is Symbol(impl) / the "impl" symbol?                                                                        
                  
  const implSymbol = Object.getOwnPropertySymbols(window.location).find(                                              
    (s) => s.description === "impl",                                                                                  
  ) as symbol;                                                                                                        
                                                                                                                      
  To answer this we need two things: what a Symbol is, and how jsdom uses it.                                         
                  
  Symbols, briefly                                                                                                    
                  
  A Symbol is a unique, unforgeable property key. Regular properties use strings (obj.foo, obj["foo"]). But you can   
  also create a property whose key is a symbol — that property becomes invisible to normal iteration (it won't show up
   in Object.keys(obj) or for...in).                                                                                  
                  
  const hidden = Symbol("secret");
  const obj = { visible: 1, [hidden]: 2 };                                                                            
  Object.keys(obj);                      // ["visible"]
  Object.getOwnPropertySymbols(obj);     // [Symbol(secret)]  ← this is how you find them                             
                                                                                                                      
  Each symbol has an optional description (the string you pass to Symbol(...)). Two symbols with the same description 
  are still distinct — but the description is still useful for finding the one you want.                              
                                                                                                                      
  How jsdom uses it

  jsdom's window.location is the user-facing object — it's the facade defined by the WHATWG spec. But the actual      
  implementation (URL parsing, navigation, the inner state) lives on a separate internal object — a LocationImpl
  instance. jsdom connects the two by attaching the impl object to the facade under a hidden symbol key whose         
  description is "impl":

  // jsdom does (conceptually):
  const implSymbol = Symbol("impl");                                                                                  
  window.location[implSymbol] = new LocationImpl(...);
                                                                                                                      
  When user code does window.location.href = "/login", the facade's setter internally does this[implSymbol].href =    
  value — delegating to the impl object.                                                                              
                                                                                                                      
  Why jsdom does this: it keeps internal machinery hidden from user code (you can't enumerate or accidentally touch   
  it), while still letting the facade reach it.
                                                                                                                      
  Why the test needs it: the facade's href setter is locked non-configurable by the WHATWG spec (jsdom enforces this  
  exactly). But the impl object's href setter on its prototype is configurable. So we dig through the symbol to find
  the impl, get its prototype, and swap in our jest mock there. The facade keeps using its setter, which keeps        
  delegating to the impl, which now hits our mock.

  Object.getOwnPropertySymbols(obj) returns all symbol-keyed properties on an object. We find the one whose           
  description is "impl". That's the key jsdom uses.
                                                                                                                      
  4. What does jest.fn<void, [string]>() mean?

  const hrefSetter = jest.fn<void, [string]>();
                                                                                                                      
  This is generic syntax. jest.fn takes two type parameters that describe the signature of the mock function:         
                                                                                                                      
  jest.fn<TReturn, TArgs extends any[]>()                                                                             
  //       ^^^^^^  ^^^^^^^^^^^^^^^^^^^                                                                                
  //       what it  the types of its
  //       returns  arguments, as a tuple                                                                             
                                                                                                                      
  So jest.fn<void, [string]>() means:                                                                                 
  - void — the mock returns nothing (mimics a setter)                                                                 
  - [string] — a tuple of argument types. One argument, of type string.                                               
                                                                       
  The square brackets wrapping string are important. They don't mean "array of strings" — they mean a fixed tuple of  
  exactly one element typed as string. TypeScript distinguishes:                                                      
                                                                                                                      
  ┌───────────────────────┬─────────────────────────────────────────────────────────────────────────────────────┐     
  │        Syntax         │                                       Meaning                                       │
  ├───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────┤
  │ string[]              │ array of any number of strings (the single arg of the mock would be a string array) │
  ├───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────┤
  │ [string]              │ exactly one argument, which is a string                                             │     
  ├───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────┤
  │ [string, number]      │ exactly two args: a string, then a number                                           │     
  ├───────────────────────┼─────────────────────────────────────────────────────────────────────────────────────┤     
  │ [string, ...number[]] │ first arg string, then any number of numbers                                        │
  └───────────────────────┴─────────────────────────────────────────────────────────────────────────────────────┘     
                  
  Tuples are how you describe an argument list, because functions have an ordered fixed-arity signature, not an array.
                  
  So the mock is typed as (v: string) => void — a single-string-arg function returning nothing. This matches the href 
  setter's signature exactly, and TypeScript will warn you if you call hrefSetter() with no args or with a number.
                                                                                                                      
  5. What is a "prototype" (in the context of Object.getPrototypeOf)?                                                 
   
  Every JavaScript object has an invisible link to another object called its prototype. When you access a property on 
  an object and it doesn't have that property directly, JS walks up through the prototype chain looking for it.
                                                                                                                      
  A concrete mental model

  class Dog {
    bark() { console.log("woof"); }
  }                                                                                                                   
   
  const rex = new Dog();                                                                                              
  rex.bark();  // works

  rex is an instance. Where does bark() live? Not on rex itself — it lives on Dog.prototype. When you write           
  rex.bark():
                                                                                                                      
  1. JS checks: does rex have an own property bark? → No.                                                             
  2. JS follows the prototype link: does Dog.prototype have bark? → Yes.
  3. JS calls it.                                                                                                     
                  
  Object.getPrototypeOf(rex) returns Dog.prototype. That's the object sitting one step up the chain.                  
                  
  Object.getPrototypeOf(rex) === Dog.prototype;  // true                                                              
                  
  Why this matters for our test                                                                                       
   
  const locationImplProto = Object.getPrototypeOf(                                                                    
    (window.location as unknown as Record<symbol, object>)[implSymbol],
  );                                                                                                                  
   
  (window.location)[implSymbol] is the impl instance — it's a specific LocationImpl object. Object.getPrototypeOf(...)
   climbs one step up and returns LocationImpl.prototype — the shared object where all the class methods and accessors
   live.                                                                                                              
                  
  Why target the prototype instead of the instance? Two reasons:                                                      
   
  1. The href setter lives there — class accessors (get href() {}, set href(v) {}) are defined on the prototype by    
  class syntax, not on each instance. So to override the setter, we go where it actually lives.
  2. The prototype is configurable, unlike the facade's own href property (which is locked per WHATWG spec). We can   
  redefine href on LocationImpl.prototype, and all LocationImpl instances now see our override.                       
   
  Visual chain for the test                                                                                           
                  
  window.location                   ← the facade (own 'href' — non-configurable, can't touch)                         
      │                                                                                                               
      │ [Symbol("impl")] →  LocationImpl instance  ← jsdom's internal impl
      │                         │                                                                                     
      │                         │ [[Prototype]] →  LocationImpl.prototype  ← OUR TARGET
      │                                               │                                                               
      │                                               │  get href() {...}
      │                                               │  set href(v) {...}   ← we replace these                       
                                                                                                                      
  When user code does window.location.href = "/login":                                                                
  1. The facade's own setter runs → it does this[implSymbol].href = "/login"                                          
  2. impl.href = "/login" — impl doesn't have own href, so JS walks up                                                
  3. Finds href setter on LocationImpl.prototype — which is now our jest.fn
  4. Our mock records the call with argument "/login"                                                                 
                                                                                                                      
  That's how we intercept a non-configurable property without jsdom's spec compliance getting in the way.             
                                                                                                                      
  ---                                                                                                                 
  TL;DR of all five                                                                                                   
                                                                                                                      
  - ResponseHandler — our hand-written type describing axios's internal {fulfilled, rejected} handler entry
  - as unknown as — TypeScript's two-step force-cast to bypass the compile-time "types don't overlap" check           
  - "impl" — jsdom stores the real Location implementation under a hidden Symbol("impl") key on the facade; we dig it 
  out to reach the configurable parts                                                                                 
  - jest.fn<void, [string]>() — a mock function typed as (arg: string) => void; the [string] is a tuple (argument     
  list), not an array                                                                                                 
  - prototype — the object one step up the inheritance chain; class methods/accessors live there, and we swap our mock
   in there because the target setter is defined on the prototype and is configurable there  