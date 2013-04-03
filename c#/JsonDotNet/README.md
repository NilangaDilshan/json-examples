The C# example code is located in a [Git repository here](https://github.com/SpotterRF/json-examples/tree/master/c%23/JsonDotNet/).

This code requires the following dependencies:

  * [JSON.NET](http://james.newtonking.com/pages/json-net.aspx)- v4.5r5 [Download Page](http://json.codeplex.com/releases/view/87440#)
    * [JSON.NET Documentation](http://james.newtonking.com/projects/json/help/)

This depends on the project files being at:
- JSON.Net: '`./Json45r5/Source/Src/Newtonsoft.Json`'

### Compile and Run the example code (with Mono Develop)

    wget 'http://download.codeplex.com/Download?ProjectName=json&DownloadId=376473&FileTime=129809414388100000&Build=18857' \
      -O Json45r5.zip
    unzip Json45r5.zip -d Json45r5

    pushd Json45r5/Source/Src/Newtonsoft.Json/
    xbuild Newtonsoft.Json.csproj 
    popd

    xbuild poll.csproj

    mono ./bin/Debug/poll.exe 169.254.254.254 # IP of spotter

To install, compile using Visual Studio, Mono Develop, or xbuild:

    xbuild poll.csproj

The poll example will:

- get the `/geolocation.json` resource from a Spotter
- grab the '`altitude`' value and do some simple modifications to it
    - example using JSON.NET
- post the changed JSON data back to the Spotter

### Compile and Run the example code (with Visual Studio)

  1. Install [NuGet](http://nuget.org/), Microsoft's own "NPM" analogue for C#
  2. Install [Json.NET via NuGet](http://nuget.org/packages/Newtonsoft.Json),
run the following command in the [Package Manager Console](http://docs.nuget.org/docs/start-here/using-the-package-manager-console)

        PM> Install-Package Newtonsoft.Json

Visual Studio makes opening and editing projects easy. Simply open the `.sln` file.

### Code Walkthrough ###

There are four major functions:

- `GetSettings` - example of making an HTTP GET request to get data
- `SetSettings` - example of making an HTTP POST request to set data
- `ParseSettingsFastJSON` - example of using the fastJSON library
- `ParseSettingsJSONNet` - example of using the JSON.NET library

Both HTTP functions use built-in libraries.

#### GetSettings ####

Takes a string representing a URL without a path.

    // create the request object
    HttpWebRequest req = (HttpWebRequest)WebRequest.Create(url + "?minWait=1000");

    // get a response
    HttpWebResponse res = (HttpWebResponse)req.GetResponse();

This sets up and makes the request. Notice that we were able to put query parameters directly into the URL. Since these are just settings files, the query parameters don't make much sense, but this is how to append them.

The call to `req.GetResponse()` blocks until the server responds. Once the response is received, we grab the stream and all of the data in it:

    // get a stream to the response body
    Stream resStream = res.GetResponseStream();

    // set up the variables we'll need
    int count = 0;
    byte[] buf = new byte[8192];
    StringBuilder sb = new StringBuilder();
    while ((count = resStream.Read(buf, 0, buf.Length)) > 0) {
        sb.Append(Encoding.UTF8.GetString(buf, 0, count));
    }

Once all of the data is received, we just pass this back.

#### GetSettingsGzip ####

This is the same as GetSettings, except it sends the additional `Accept-Encoding` header letting the server know that we prefer gzipped data:

    req.Headers.Add(HttpRequestHeader.AcceptEncoding, "gzip;q=1.0");

Then, if the `Content-Encoding` header is set to gzip, we create an instance of  [`GZipStream`](http://msdn.microsoft.com/en-us/library/system.io.compression.gzipstream.aspx) (from `System.IO.Compression`). Since it is a `Stream`, we just replace our local variable that stores the response stream with it:

    Stream resStream = res.GetResponseStream();
    if (res.ContentEncoding.Equals("gzip", StringComparison.CurrentCultureIgnoreCase)) {
        resStream = new GZipStream(resStream, CompressionMode.Decompress);
    }

That's it! We can leave the other code the same. If it's not gzipped, it will be handled as before.

#### SetSettings ####

Sending the settings up to the server is quite similar, but we'll need to set more things on the request. In this example, `settings` is the result from `GetSettings`.

    HttpWebRequest req = (HttpWebRequest)WebRequest.Create(url + RESOURCE + "/settings");
    req.Method = "POST";
    req.ContentType = "application/json";
    req.ContentLength = settings.Length;

Here, we specify that this is a POST request, whose content is JSON encoded. This is very important, because the Spotters only respond to data that is specifically stated as being JSON. We go ahead and set the content length because we know it.

Next, we write the data to the request stream:

    // get the request stream and write our data back to it
    Stream writeStream = req.GetRequestStream();
    writeStream.Write(bytes, 0, bytes.Length);

And finally get the response:

    HttpWebResponse res = (HttpWebResponse)req.GetResponse();

Once we have the response, we grab the data again as before:

    Stream resStream = res.GetResponseStream();

    StringBuilder sb = new StringBuilder();
    byte[] buf = new byte[8192];
    int count = 0;
    while ((count = resStream.Read(buf, 0, buf.Length)) > 0) {
        sb.Append(Encoding.UTF8.GetString(buf, 0, count));
    }

The response tells us if the request succeeded, or if it failed.

#### ParseSettingsFastJSON ####

In this example, we'll make a lot of assumptions about the structure of the data. In practice, more care should be taken in ensuring that the data is well formed and as we expect.

Objects in `fastJSON` are represented as a `Dictionary<string, object>`. To start off, we'll need to parse our object:

    Dictionary<string, object> jsonData = fastJSON.JSON.Instance.Parse(settings) as Dictionary<string, object>;

From here on out, it's just like working with a regular Dictionary. To start, grab the `result` object that has all of our settings:

    Dictionary<string, object> result = jsonData["result"] as Dictionary<string, object>;

Grab the altitude and manipulate it:

    float alt = float.Parse(result["altitude"] as string);
    if (alt > 2000) {
        alt = 747;
    } else {
        alt += 1000;
    }

Then put it back in:

    result["altitude"] = alt;

Then convert it back to a string:

    JSON.Instance.ToJSON(result);

This can now be sent back to the server! Notice that we're only sending back the result part of it. The header should be discarded.

#### ParseSettingsJSONNet ####

In this example, we'll make a lot of assumptions about the structure of the data. In practice, more care should be taken in ensuring that the data is well formed and as we expect.

To start off, we'll parse our settings into an object:

    JObject obj = JObject.Parse(settings);

Then grab the result object inside:

    JToken result = obj["result"];

Now that we have our result object, let's grab the altitude and manipulate it:

    float alt = result["altitude"].Value<float>();
    if (alt > 2000) {
        alt = 747;
    } else {
        alt += 1000;
    }

Now that we're done, we'll put it back in:

    result["altitude"] = alt;

Turning this back into a string is as simple as calling it's ToString() method:

    result.ToString();

That's it! It can now be sent to the server. Notice that we're only sending back the result part of it--the header should be discarded.

