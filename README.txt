** PROJECT README **
**not allowed to upload code becuase that's probably not ok in a company.

hello. I am writing this as a method of documentation and procrastination 
because I do not want to move on to the next project. 

Admin Library:
	handles all admin side of the server requests and
	currently outputs strings holding the response and the body of data.
User Library:
	handles all user activity, which is honesetly the bulk of it

In the process of learning and making a dll I ran into 3 big things:
	- cpprestSDK
	- making the dll
	- uploading / downloading binary data


===========================================================================
                              CPPRESTSDK
===========================================================================

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



=========================================================================================
                	GENERAL STRUCTURE OF ANY GIVEN FUNCTION:
=========================================================================================

**these functions heavily use cpprestsdk (casablanca)**
http_client client(baseuri);								// Create http_client to send the request.
http_request request(methods::GET);							// build body of request
															// - methods::GET | methods::POST | methods::PATCH | methods::DEL
uri_builder builder(conversions::to_string_t(				// builds the rest of the url that you need for the request
	"/models/" + std::to_string(modelID) + 
	"/attributes"));
request.set_request_uri(builder.to_uri());					// sets request
if (!m_szSessionToken.empty())								// a session header 
	request.headers().add(L"Cookie", m_szSessionToken);		// - cookies keep track of who you are in your session
request.headers().add(L"Content-Type", 						// specify content type for requests with data in body:
	L"application/x-www-form-urlencoded");		
string str = "name=" + name + "&value=" + value;			// build the string in urlencoded form
request.set_body(str);										// set the body
try {														// send request (or at least try)
	auto task = client.request(request);					// send final request
	while (!task.is_done());								// wait for async task to finish, that way we know all the data is here
	return response_to_string(task.get());					// response_to_string is a helper function that displays all relevant data
}
catch (const std::exception &e) {							// error handling
	printf("Error exception:%s\n", e.what());
}
return "";



=============================================================================
                			UPLOAD / DOWNLOAD
=============================================================================

UPLOAD:
- data uploads are in the format of multipart/form-data. 
- I use a library to handle the full format. 
	- this is because the body of an upload request
	  uses the multipart/form-data format.
	- parser.addFile
		- add file data to be uploaded. 
		- takes the name (hardcoded as "file")
		- takes filename (full location to be specific)
	- parser.boundary
		- boundaries seperate one file's data from the next
		  this allows us to upload several files at once
		- a randomly generated boundary is made, heightens security
		- pass this boundary into the header to allow the server to 
		  be able to decode all of the data
	- parser.GenBodyContent
		- returns full body in string form, to be set in request.set_body()


---------------------------- code start -----------------------------------------
MultipartParser parser;                   			 	// declare parser
for (int i = 0; i < size; i++)		    				// add files, the field is "files"
	parser.AddFile("file", files[i]); 					// -files[] is an array of file locations on machine.
std::string boundary = parser.boundary();			    // save randomly genarated boundary. 
														// - needed for server to decode. 
std::string body = parser.GenBodyContent(); 			// generate full body in string form

request.set_body(body, 									// set body, specifying content type
	"multipart/form-data; boundary=" + boundary);


---------------------------- code end -----------------------------------------

DOWNLOAD
- the body of the request is raw byte data. we must take all of this binary data
  and write it into a file. 


---------------------------- code start-----------------------------------------

auto byte_vector = response.extract_vector().get(); 	// returns a vector containing byte data
std::ofstream output_file(filename,						// prepares the output file. 
	std::ios::out | std::ios::binary);					// - filename ripped from header of response. 
														// - ('Content-Disposition')

output_file.write((const char*)&byte_vector[0], 		// writing all bytes into vector. 
	byte_vector.size());								// - file.write() needs type (const char*)
output_file.close(); 									// closing files like a good coder. 


---------------------------- code end -----------------------------------------



===========================================================================
                                 DLL
===========================================================================

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
		- Macro: 'xcopy /y /d "..\..\YourLibrary\$(IntDir)YourLibrary.dll" "$(OutDir)"'
		- add a command that copies the dll to your build output directory
		- that way the system can find the dll if it's not anywhere else in the directory
		- I added the cpprest_2_10d.ddl in the same folder to appease the dll finding gods
			(debug dll)
		- dependencies needed in DEBUG folder:
			cpprest_2_10d.dll
			boost_date_time-vc141-mt-gd-x32-1_67.dll
			boost_system-vc141-mt-gd-x32-1_67.dll
			zlibd1.dll
			UserLibrary.dll *OR* AdminLibrary.dll		

good luck brother. I wish for all your DLL's to compile.

