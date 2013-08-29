ArcGIS.PCL
==========

Use ArcGIS Server REST resources without an official SDK [more information](http://davetimmins.wordpress.com/2013/07/11/arcgis-pclthe-what-why-how/).

It can also be used for just working with types and as well as some ArcGIS Server types you can also use GeoJSON FeatureCollections with the ability to convert GeoJSON <-> ArcGIS Features.

Typical use case would be the need to call some ArcGIS REST resource from server .NET code or maybe a console app. Rather than having to fudge a dependency to an existing SDK you can use this. 
Should work with .NET for Windows Store apps, .NET framework 4.5, Silverlight 4 and higher, Windows Phone 7.5 and higher

Since the serialization is specific to your implementation you will need to create an ISerializer to use in your gateway. The test project has ServiceStack.Text and Json.NET [example serializers](https://github.com/davetimmins/ArcGIS.PCL/blob/dev/ArcGIS.Test/ISerializer.cs) 

Supports the following as typed operations:

 - Generate Token (automatically if credentials are specified in gateway)
 - Query
 - Apply Edits
 - Single Input Geocode
 - Reverse Geocode
 - Describe site (returns a url for every service)
 - Simplify
 - Project

Some example of it in use for server side processing in web sites

 - [Describe site] (https://arcgissitedescriptor.azurewebsites.net/)
 - [Convert between GeoJSON and ArcGIS Features] (http://arcgisgeojson.azurewebsites.net/)
 - [Server side geometry operations] (http://eqnz.azurewebsites.net/)
 - [Server side geocode] (http://loc8.azurewebsites.net/map?text=wellington, new zealand)

See some of the [tests](https://github.com/davetimmins/ArcGIS.PCL/blob/dev/ArcGIS.Test/ArcGISGatewayTests.cs) for some example calls.

###Gateway Use Cases

#### ArcGIS Server with non secure resources
```csharp
public class ArcGISGateway : PortalGateway
{
    public ArcGISGateway(ISerializer serializer)
        : base(@"http://sampleserver3.arcgisonline.com/ArcGIS/", serializer)
    { }
}
```
#### ArcGIS Server with secure resources
```csharp
public class SecureGISGateway : SecureArcGISServerGateway
{
    public SecureGISGateway(ISerializer serializer)
        : base(@"http://serverapps10.esri.com/arcgis", "user1", "pass.word1", serializer)
    { }
}
```
#### ArcGIS Server with secure resources and token service at different location
```csharp
public class SecureTokenProvider : TokenProvider
{
    public SecureTokenProvider(ISerializer serializer)
        : base(@"http://serverapps10.esri.com/arcgis", "user1", "pass.word1", serializer)
    { }
}

public class SecureGISGateway : PortalGateway
{
    public SecureGISGateway(ISerializer serializer, ITokenProvider tokenProvider)
        : base(@"http://serverapps10.esri.com/arcgis", serializer, tokenProvider)
    { }
}
```

#### ArcGIS Online either secure or non secure  
```csharp
public class ArcGISOnlineGateway : PortalGateway
{
    // non secure access
    public ArcGISOnlineGateway(ISerializer serializer)
        : base(PortalGateway.AGOPortalUrl, serializer, null)
    { }

    // secure access
    public ArcGISOnlineGateway(ISerializer serializer, ArcGISOnlineTokenProvider tokenProvider)
        : base(PortalGateway.AGOPortalUrl, serializer, tokenProvider)
    { }
}
```
### Converting between ArcGIS Feature Set from hosted FeatureService and GeoJSON FeatureCollection
```csharp
static ISerializer _serializer = new ServiceStackSerializer();
static Dictionary<String, Func<String, FeatureCollection<IGeoJsonGeometry>>> _funcMap = new Dictionary<String, Func<String, FeatureCollection<IGeoJsonGeometry>>>
{
    { GeometryTypes.Point, (uri) => new ProxyGateway(uri, _serializer).GetGeoJson<Point>(uri) },
    { GeometryTypes.MultiPoint, (uri) => new ProxyGateway(uri, _serializer).GetGeoJson<MultiPoint>(uri) },
    { GeometryTypes.Envelope, (uri) => new ProxyGateway(uri, _serializer).GetGeoJson<Extent>(uri) },
    { GeometryTypes.Polygon, (uri) => new ProxyGateway(uri, _serializer).GetGeoJson<Polygon>(uri) },
    { GeometryTypes.Polyline, (uri) => new ProxyGateway(uri, _serializer).GetGeoJson<Polyline>(uri) }
};

...

var layer = new ProxyGateway(uri, _serializer).GetAnything(uri.AsEndpoint());
if (layer == null || !layer.ContainsKey("geometryType")) throw new HttpException("You must enter a valid layer url.");
return _funcMap[layer["geometryType"]](uri);

...

public class AgsObject : JsonObject, IPortalResponse
{
    [System.Runtime.Serialization.DataMember(Name = "error")]
    public ArcGISError Error { get; set; }
}

public class ProxyGateway : PortalGateway
{
    public ProxyGateway(String rootUrl, ISerializer serializer)
        : base(rootUrl, serializer)
    { }

    public QueryResponse<T> Query<T>(Query queryOptions) where T : IGeometry
    {
        return Get<QueryResponse<T>, Query>(queryOptions).Result;
    }

    public AgsObject GetAnything(ArcGISServerEndpoint endpoint)
    {
        return Get<AgsObject>(endpoint).Result;
    }

    public FeatureCollection<IGeoJsonGeometry> GetGeoJson<T>(String uri) where T : IGeometry
    {
        var result = Query<T>(new Query(uri.AsEndpoint()));
        result.Features.First().Geometry.SpatialReference = result.SpatialReference;
        var features = result.Features.ToList();
        if (result.SpatialReference.Wkid != SpatialReference.WGS84.Wkid)
            features = new ProjectGateway(Serializer).Project<T>(features, SpatialReference.WGS84);
        return features.ToFeatureCollection();
    }
}

```
### Converting between GeoJSON FeatureCollection and ArcGIS Feature Set
```csharp
static Dictionary<String, Func<String, List<Feature<IGeometry>>>> _funcMap = new Dictionary<String, Func<String, List<Feature<IGeometry>>>>
{
    { "Point", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonPoint>>(data).ToFeatures<GeoJsonPoint>() },
    { "MultiPoint", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonLineString>>(data).ToFeatures<GeoJsonLineString>() },
    { "LineString", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonLineString>>(data).ToFeatures<GeoJsonLineString>() },
    { "MultiLineString", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonLineString>>(data).ToFeatures<GeoJsonLineString>() },
    { "Polygon", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonPolygon>>(data).ToFeatures<GeoJsonPolygon>() },
    { "MultiPolygon", (data) => JsonSerializer.DeserializeFromString<FeatureCollection<GeoJsonMultiPolygon>>(data).ToFeatures<GeoJsonMultiPolygon>() }
};
```

### Download
If you have [NuGet](http://nuget.org) installed, the easiest way to get started is to install via NuGet:

    PM> Install-Package ArcGIS.PCL

or you can get the code from here.