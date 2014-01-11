---
layout: post
title: "Network Stubs with OHHTTPStubs"
date: 2014-01-10 19:42
comments: true
categories: ios kiwi kif objc ohhttpstubs
---
This week I found myself implementing the Google Places' API in an iOS application, what better occasion to write a post about my favourite iOS stub framework, [OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs)?
<!-- More -->
#####Network stubs
There are a lot of interesting [articles](http://martinfowler.com/articles/mocksArentStubs.html) that explain the philosophy behind stubbing, mocking and their differences. I like the concise definition given by [Gregg Pollack](https://twitter.com/greggpollack) in [Codeschool's Rails Testing for Zombies](http://www.codeschool.com/courses/rails-testing-for-zombies):
{% blockquote %}
Stubs replace a method with code that returns a specified result, mocks are stubs with an assertion that the method gets called.
{% endblockquote %}
That being said, when testing network code, stubs are _really_ useful tools to avoid hitting a network resource each time our tests run. This prevents unwanted traffic (and that's nice when you are dealing with API limits) and speeds up our test suite quite a bit.

#####Google Places API
I won't go further in detail with the Google Places' API, what we need to know is the API's URL and the format of returned JSON object.
{% codeblock API's URL %}
https://maps.googleapis.com/maps/api/place/nearbysearch/json?key=<your_api_key>&location=<lat,lng>&sensor=true&radius=500
{% endcodeblock %}

{% codeblock %}
{
	debug_info: [ ],
	html_attributions: [ ],
	next_page_token: "ClRHAAAAFwvsXH5hYznZdQOQ61ZCp7gLZmcRVjy3l5OadZe4gdJNrrZb9xocYXTVHnSM3HKjQ41oBM0j7ImXN21Z6guuGMhpRg9fNOlZbN8vHPp1woYSEHZreMp6Y0xJxlpBYpbcslUaFIQravjfeq49dBvwPHGUQzsQwa9p",
	results: [
		{
		geometry: {
			location: {
				lat: 37.7815533,
				lng: -122.4156427
			},
			viewport: {
				northeast: {
					lat: 37.7828015,
					lng: -122.4125167
				},
				southwest: {
					lat: 37.7741122,
					lng: -122.4222884
				}
			}
		},
		icon: "http://maps.gstatic.com/mapfiles/place_api/icons/geocode-71.png",
		id: "b7fa5ebba26a10513d18e1fd50628f082ffbd1a2",
		name: "Civic Center",
		reference: "CrQBoQAAAHvUxRoSt-y3wsF3Cr6JgGO5Y4Q8nWfthCZLFnApqPO9xmy1NnuQnaZ6BqqE9XPbx8rNKBF-IR7R8C8-8O90pkHjcmoUZKtZscZwbuTYa6RLO5ILIl34AGTh8lbB1cdPtt_NXbkQTQg8eiptx_gzYH8BKg8AKWhYFCN5u9xaE9lXENILw2Ngw_TaUoz1DAAwL6s7uLP3nTQIsO5tOVdsGh3dv2F4ZPck2dHBqD3gPHLSEhArD0KzT3KhPbFWq9UII33pGhSBLiwjfI7y3uOwGS8pkCskS6RK8Q",
		types: [
			"neighborhood",
			"political"
		],
		vicinity: "San Francisco"
		}
	]
	status: "OK"
}
{% endcodeblock %}
Great, we have a lot of info, but I really need just ```name``` and ```id``` in this project.

#####Writing the test
Ok, we figured out how to ask Google for directions and we have our JSON response, next up is our test.

In the last blog post I wrote about my iOS test configuration with Specta. In this project though I'll be using [Kiwi](https://github.com/allending/Kiwi), nothing mayor really changes, just the syntax.

Here's the spec:
{% codeblock lang:objc %}
describe(@"fetchSuggestionsForLocation:onSuccess:onFailure", ^{
	context(@"with valid data", ^{
		it(@"returns an array", ^{
			__block NSArray* result;
			[[AMPlacesHelper sharedHelper] fetchSuggestionsForLocation:(CLLocationCoordinate2D){37.7749295,-122.4194155} onSuccess:^(NSArray *data) {
				result = data;
			} onFailure:^(NSError *error) {}];
			[[expectFutureValue(result) shouldEventually] beKindOfClass:[NSArray class]];
			[[expectFutureValue(theValue(result.count)) shouldEventually] beGreaterThan:theValue(0)];
		});
		
		it(@"returns a JSON array of places with a name and id", ^{
			__block NSArray* result;
			[[AMPlacesHelper sharedHelper] fetchSuggestionsForLocation:(CLLocationCoordinate2D){37.7749295,-122.4194155} onSuccess:^(NSArray *data) {
				result = data;
			} onFailure:^(NSError *error) {}];
			[[expectFutureValue(result[0][@"name"]) shouldEventually] beNonNil];
			[[expectFutureValue(result[0][@"id"]) shouldEventually] beNonNil];
		});
	});
	
	context(@"with invalid data", ^{
		it(@"returns an error in the failure block", ^{
			__block NSError* resultError;
			[[AMPlacesHelper sharedHelper] fetchSuggestionsForLocation:(CLLocationCoordinate2D){37.7749295,-122.4194155} onSuccess:^(NSArray *data) {} onFailure:^(NSError *error) {
				resultError = error;
			}];
			[[expectFutureValue(resultError) shouldEventually] beNonNil];
		});
	});	
});
{% endcodeblock %}

Pretty basic stuff, the first test checks that the returned object is an ```NSArray```, and that its content's lenght is greater than 0. The second test checks for the content itself, making sure that ```id``` and ```name``` are present. The last spec just checks against invalid data, making sure that an error is raised. ```expectFutureValue``` waits 1 seconds (by default) before raising the expectation. This is key when dealing with asyncronous calls.
 
You may point out that it's always a good practice to limit expectations to one per spec, but since these are pretty basic, I figured I could get away with squeezing two of them in the same spec.

Running the test with my trusty xctool script, I see 3 red specs, yay!

#####Stubbing the network
Now we could implement our code and run the test again, hoping for green, but once we manage to make the network call, we'll be hitting the Google Places' API once for every test run. That's bad, so here enters [OHHTTPStubs](https://github.com/AliSoftware/OHHTTPStubs).

OHHTTPStubs is pretty cool, it lets you register a stub that will listen for any network request and respond with a preset response body and response code. This means that we can easily emulate the network API's behaviour and use it to our likings.
The basic structure of a stub is this one:
{% codeblock lang:objc %}
[OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
	// Here you can decide whether to stub the request or not, based for example on the request URL
	return YES;
} withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
	// Here you return the fake data from your stubbed network call
	return [OHHTTPStubsResponse responseWithJSONObject:@[ @"hello" ] statusCode:200 headers:@{@"Content-Type": @"application/json"}];
}];
{% endcodeblock %}
Pretty nifty. Once we described our test we can then tear down the stubs in an ```afterAll``` block:

{% codeblock lang:objc %}
afterAll(^{
    [OHHTTPStubs removeAllStubs];
});
{% endcodeblock %}

#####Our fixture
Since I want to stub the Google Places' API I need to provide a sort of fixture throught OHHTTPStubs. Let's ```curl``` the result, and save it to a JSON file that will be served by the stub.

{% codeblock %}
curl https://maps.googleapis.com/maps/api/place/nearbysearch/json?key=<your_api_key>&location=<lat,lng>&sensor=true&radius=500 > google_places.json
{% endcodeblock %}
Let's put this file in the test bundle of our iOS application and write the stub for the context ```with valid data```:

{% codeblock lang:objc %}
beforeEach(^{
	[OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
		return YES;
	} withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
		NSBundle *bundle = [NSBundle bundleForClass:[self class]];
		return [OHHTTPStubsResponse responseWithFileAtPath:[bundle pathForResource:@"google_places" ofType:@"json"]
												statusCode:200
												   headers:@{@"Content-Type": @"application/json"}];
	}];
});
{% endcodeblock %}

And for the context ```with invalid data```:
{% codeblock lang:objc %}
beforeEach(^{
	[OHHTTPStubs stubRequestsPassingTest:^BOOL(NSURLRequest *request) {
		return YES;
	} withStubResponse:^OHHTTPStubsResponse*(NSURLRequest *request) {
		return [OHHTTPStubsResponse responseWithData:nil statusCode:500 headers:nil];
	}];
});
{% endcodeblock %}

We should be set, let's make sure that our test fail in a meaningful way:
{% codeblock %}
> xctool -workspace Project.xcworkspace -scheme 'Project' -sdk iphonesimulator test
...
'AMPlacesHelper, fetchSuggestionsForLocation：onSuccess：onFailure, with valid data, returns an array' [FAILED], expected subject to be kind of NSArray, got (null)
'AMPlacesHelper, fetchSuggestionsForLocation：onSuccess：onFailure, with valid data, returns a JSON array of places with a name and id' [FAILED], expected subject not to be nil
'AMPlacesHelper, fetchSuggestionsForLocation：onSuccess：onFailure, with invalid data, returns an error in the failure block' [FAILED], expected subject not to be nil
{% endcodeblock %}
Nice! We can now implement the code that will let the test pass, but won't hit the network.

#####From red to green
Let's implement the code that will pass our test. As always, I used [AFNetworking](https://github.com/AFNetworking/AFNetworking) to do a quick GET request to the aforementioned API.

{% codeblock lang:objc %}
- (void)fetchSuggestionsForLocation:(CLLocationCoordinate2D)coordinates onSuccess:(void(^)(NSArray* data))success onFailure:(void(^)(NSError* error))failure
{
	[[UIApplication sharedApplication] setNetworkActivityIndicatorVisible:YES];
	AFHTTPRequestOperationManager *manager = [AFHTTPRequestOperationManager manager];
	NSDictionary* params = @{@"key": kGooglePlacesKey,
							 @"location": [NSString stringWithFormat:@"%f,%f", coordinates.latitude, coordinates.longitude],
							 @"sensor": @"true",
							 @"radius": @"500"};
	[manager GET:kGooglePlacesURL parameters:params success:^(AFHTTPRequestOperation *operation, id responseObject) {
		[[UIApplication sharedApplication] setNetworkActivityIndicatorVisible:NO];
		if (success) {
			success(responseObject[@"results"]);
		}
	} failure:^(AFHTTPRequestOperation *operation, NSError *error) {
		[[UIApplication sharedApplication] setNetworkActivityIndicatorVisible:NO];
		if (failure) {
			if (error) {
				failure(error);
			} else {
				failure([[NSError alloc] initWithDomain:@"googleapi.com" code:500 userInfo:@{@"message": @"unable to retrieve places"}]);
			}
		}
	}];
}
{% endcodeblock %}

We run our suite again:
{% codeblock %}
** TEST PASSED: 3 passed, 0 failed, 0 errored, 3 total **
{% endcodeblock %}
We're green. We can turn off Wifi and unplug the ethernet cable, the test will pass anyway.

If we want to test the code with the live feed, we can easily switch the return value for ```stubRequestsPassingTest``` in our stubs to ```NO```.

#####Debugging stubs
When using OHHTTPStubs there's one caveat...  

While writing the stub, I did manage to sneak a typo in my stub code, so I was trying to load ```google.places.json``` instead of ```google_places.json```. Usually you'd find this error pretty easilly, but this time I only noticed that every stubbed spec that was previously green, now was failing with this generic error:
{% codeblock %}
Test did not run: the test bundle stopped running or crashed in AMPlacesHelper_FetchSuggestionsForLocationonSuccessonFailure_WithValidData_ReturnsAnArray
{% endcodeblock %}
This can be annoying to debug if you have complex stub and I didn't find any quick solution, beside being careful when writing the stub implementation. I guess that a good rule of thumb here is: 
{% blockquote %}
keep your stubs as simple as possible
{% endblockquote %}
This should really apply to every good stub and mock.

Until next time.  
Andrea