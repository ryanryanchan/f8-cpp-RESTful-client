** PROJECT README **
**not allowed to upload code becuase that's probably not ok in a company.

hello. I am writing this as a method of documentation and procrastination 
because I do not want to move on to the next project. 

the the admin library handles all admin side of the server requests and
currently outputs strings holding the response and the body of data.

in the process of learning and making a dll I ran into 2 big things:
	- cpprestSDK
	- making the dll


// ====================================================================
//                           CPPRESTSDK
// ====================================================================

This library uses a library called cpprestsdk (casablanca). 
documentation here: 
	http://microsoft.github.io/cpprestsdk/namespaces.html

there are many namespaces and types.
here are the important ones:
	- http_client: 
		a client created to build a request.
		this client will initially hold a base url, and takes a http_request
		object. we put it into a task, and use while(!task.is_done()) to
		wait for the function to complete. (we want to make sure all 
		asynchronous tasks finish before taking out the response)
	
	- http_request:
		A type that you can build, and then feed into the http_client.
		request(methods::GET)
			- initialization

		request.set_request_uri()	
			- sets the full uri of the request
			- builds off of baseurl
			- needs a uri type

		request.headers().add(L"Content-Type", L"application/x-www-form-urlencoded");
			- a necessary header in our code to denote encoding
			- sample body with this encoding: "name=Ryan&role=the_best"

		request.set_body(str)
			- sets the body with whatever string you feed into it

	- http_response:
		the response type that you get back from every request

		response.status_code();
			- return the status code in unsigned int form. 
			- I used std::to_string() to change it into a variable that I could process
		response.reason_phrase()
			- pretty sure it's a wstring holding a reason, but it's its own type here.
			- used the string conversion and iterators to get the string form of the reason
			- string reason_str(reason.begin(), reason.end())
		response.extract_string().get()
			- returns the body of the response in wstring
			- string response_str(response_wstring.begin(), response_wstring.end())
	
	- wstring/string_t/char_t: different typedefs used in the utility namespace. 
		you have to use utility::conversions::to_string_t(std::string)
		to put things into a uri builder
		converting from wstring to string 
			std::string new_str(wstring.begin(), wstring.end())

	auto task = client.request(request);
	while (!task.is_done());
	return response_to_string(task.get());
		- this is an important chunk of code
		- task is a concurrency task. this is an async function that we need to wait for
		- waiting is done in the while loop
		- task.get recieves a http_response object
		- response_to_string is my own function.

// ====================================================================
//                           DLL
// ====================================================================

whoooo man this one hurt to figure out. 
basically the best resource for making a dll in visual studio 2017 is here:
	 https://docs.microsoft.com/en-us/cpp/build/walkthrough-creating-and-using-a-dynamic-link-library-cpp

A DLL is a dynamically linked library. it exports variables, functions, and resources by name. 
I created a dll, but it uses the casasblanca DLL as well. 

in simple terms:
	"Unlike a statically linked library, Windows connects the imports in your app to the exports in a DLL at load time or at run time, instead of connecting them at link time. Windows requires extra information that isn't part of the standard C++ compilation model to make these connections. "

**The most important thing about DLL's is making sure the compiler knows where to look.**
Important tidbits, in no real order: (looked at website while writing this)
CREATING THE DLL
	- you gotta select DLL in the project man
	- good naming convention helps a lot when you read the outputs
		- AdminApiLibrary is obvious what it means, even when buried in a lot of garbage text
	- configuration properties > C/C++ > Preprocessor Definitions
		make sure YOURLIBRARY_EXPORTS is here
	- HEADER FILES
		- "#pragma once" is the new header guards. nice.

		#ifdef YOURLIBRARY_EXPORTS
		#define YOURLIBRARY_API __declspec(dllexport)
		#else
		#define YOURLIBRARY_API __declspec(dllimport)
		#endif
			- this is the preprocessor at work here. you need the library export file
			- the YOURLIBRARY_API macro sets '__declspec(dllimport)' modifier to the declarations. you need this so the compiler knows which functions to export to the dll file. 
			ex: YOURLIBRARY_API bool get_good_at_coding();
		- extern "C" was unneccesary for me, but it translates everything into C
			this is nice for translating to diffrerent languages, which I'm not doing
			this sucks because you need to use C. no strings. 
	- build solution: Ctrl + Shift + 'B'
		- what a timesaver
		outputs:
			1> build started
			1>stdafx.cpp
			1>YourLibrary.cpp
			1>dll.main
			1>Generating Code...
			1>	Creating Library... \your_library.lib and object \your_library.exp 
				*** YOU NEED THE LIB
			1>YourLibrary.vcxproj -> \YourLibrary.dll 
				*** YOU NEED THIS
CREATING THE CLIENT
	- most important thing is that you need to link the correct files!
	- ADD H FILE TO INCLUDE PATH
		Configuration > C/C++ > General > Additional Include Directories > 
			ADD THE FOLDER WITH YOUR .H FILE
	- ADD DLL IMPORT LIBRARY
		Configuration > Linker > Input > Additional Dependencies > 
			- add YourLibrary.lib in the top edit control
		Configuration > Linker > General > Additional Library Directories >
			- add the path to the YourLibrary.lib file
			- can use this macro '..\..\MathLibrary\$(IntDir)'
	- POST BUILD EVENT:
		- add a command that copies the dll to your build output directory
		- that way the system can find the dll if it's not anywhere else in the directory
		- I added the cpprest_2_10d.ddl here
			(debug dll)

good luck brother. I wish for all your DLL's to compile.

