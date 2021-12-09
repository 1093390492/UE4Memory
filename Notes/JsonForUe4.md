## UE4解析Json
蛮常规的一个功能点 直接看代码


从本地拿到json文件  
-

```C++
//.h

	UFUNCTION(BlueprintCallable)
	static bool ReadFileToString(const FString& filePath, FString& fileData);
-----------------------------------------
//.cpp
    bool UFunctionLibrary::ReadFileToString(const FString& filePath, FString& fileData)
    {
        if (FPaths::ValidatePath(filePath) && FPaths::FileExists(filePath))
        {
            return FFileHelper::LoadFileToString(fileData, *filePath);
        }
        return false;
    }
```
对Json的序列化与反序列化
```C++
//.h
	static bool StringToJsonObject(const FString& StringJson, TSharedPtr<FJsonObject>& JsonObject);

	static bool JsonObjectToString(const TSharedPtr<FJsonObject>& JsonObject, FString& StringJson);
--------------------------------------
//.cpp
    bool UFunctionLibrary::StringToJsonObject(const FString& StringJson, TSharedPtr<FJsonObject>& JsonObject)
    {
        if (StringJson.IsEmpty())
        {
            JsonObject = nullptr;
            return false;
        }
        
        TSharedRef<TJsonReader<>> Reader = TJsonReaderFactory<>::Create(*StringJson);
        TSharedPtr<FJsonObject> OutJsonObj = MakeShareable(new FJsonObject);
        if (FJsonSerializer::Deserialize(Reader, OutJsonObj))
        {
            JsonObject = OutJsonObj;
            return true;
        }
        else
        {
            JsonObject = nullptr;
            return false;
        }
    }

    bool UFunctionLibrary::JsonObjectToString(const TSharedPtr<FJsonObject>& JsonObject, FString& StringJson)
    {
        if (JsonObject == nullptr)
        {
            return false;
        }
        TSharedRef<TJsonWriter<>> Writer = TJsonWriterFactory<>::Create(&StringJson);
        FJsonSerializer::Serialize(JsonObject.ToSharedRef(), Writer);
        return true;
    }

```
FJsonValue的一些常用API
```C++
    //数组
    TArray<TSharedPtr<FJsonValue>> ArrayName = _jsonObject->GetArrayField("JsonArrayName");
    //常量
    FString name =_jsonObject->GetStringField("userName");
    int id = _jsonObject->GetNumberField("userId");
    //嵌套
    TSharedPtr<FJsonObject> JsonObject = _jsonObject->GetObjectField("ObjectName");
```