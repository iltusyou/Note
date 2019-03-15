#	API參數轉為Class，API參數對應

這個文章主要是在解決兩個問題

1. 將對方傳送參數與自己參數對應(也就是說對方用的參數命名可以跟你的不同，但是可以寫個方法對硬起來)
2. 傳入的參數改為一個class

比如說原本的Api的GET方法會是像這樣的寫法，傳入一個參數id，然後還有一個驗證來源用的valid_code，假設只有在valid_code=="Hello"時才算驗證成功。

```csharp
[HttpGet]
[Route("api/getArticle")]
public HttpResponseMessage GetArticle(int id, string valid_code)
{
    if(valid_code=="Hello")
    {
    
    }
}
```
但是這種寫法看起來不太喜歡，因為
1. 參數的格式不是C#常用的每個字首大寫而是用底線連接。
2. 同樣的驗證方法也會在其他地方用到，也有可能相當複雜。

而為了解決這兩個問題，我希望可以把request的參數也變成一個class，然後這個class再去繼承驗證的class，這麼一來即使有其他方法需要同樣的驗證也都不需要再寫一次了。
此外，無論是在GET或POST都可以使用同樣的class。

而在C#裡面，POST已經有既有的方法可以做到這件事了。

```csharp
/// <summary>
/// 驗證API
/// </summary>
[DataContract]
public class ValidateClass
{
    [DataMember(Name = "valid_code")]
    public string ValidCode { get; set; }

    /// <summary>
    /// 驗證
    /// </summary>
    public bool Validate()
    {        
        return this.ValidCode=="Hello";
    }
}
```
這個時候我們的request只要繼承了ValidateClass就可以使用驗證。
```csharp
[DataContract]
public class Request : ValidateClass
{
    [DataMember(Name = "id")]
    public int ID { get; set; }
}

[HttpPost]
[Route("api/getArticle")]
public HttpResponseMessage GetArticle(JObject jsonResult)
{
    Request request;
    string jsonString = jsonResult.ToString();
    using (MemoryStream ms = new MemoryStream(Encoding.Unicode.GetBytes(jsonString)))
    {
        DataContractJsonSerializer serializer = new DataContractJsonSerializer(Request));
        request = (Request)serializer.ReadObject(ms);
    }
    
    if(request.Validate())
    {
        //驗證成功
    }
}
```
但是在GET時，沒有直接做到這種事情的方法。所以我們要來寫一個ModelBinder。

```csharp
public class MemberModelBinder : IModelBinder
{
    public bool BindModel(HttpActionContext actionContext, ModelBindingContext bindingContext)
    {
        var model = Activator.CreateInstance(bindingContext.ModelType);           
        foreach (var prop in bindingContext.PropertyMetadata)
        {
            //取得要對應的屬性名稱
            var attr = bindingContext.ModelType.GetProperty(prop.Key)
            .GetCustomAttributes(false)
            .OfType<DataMemberAttribute>()
            .FirstOrDefault();
            var qsParam = (attr != null) ? attr.Name : prop.Key;

            //該屬性的值
            var value = bindingContext.ValueProvider.GetValue(qsParam) == null ?
            null :
            bindingContext.ValueProvider.GetValue(qsParam).AttemptedValue;
            var property = bindingContext.ModelType.GetProperty(prop.Key);

            if(value!=null)
            {
                //可寫入且形態相同才寫入，這邊是考慮了方法或是唯獨的情形
                if (property.CanWrite && property.PropertyType.Name == value.GetType().Name)
                    {
                        property.SetValue(model, value);
                    }
                }                             
            }
            bindingContext.Model = model;
            return true;
        }
    }
```
如此一來GET的參數也可以寫成一個class，然後也可以使用共同的方法了。

```csharp
[HttpGet]
[Route("api/getArticle")]
public HttpResponseMessage GetMember([ModelBinder(typeof(MemberModelBinder))] Request request)
{
    if(request.Validate())
    {
        //驗證成功
    }
}

```
