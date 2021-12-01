## 使用C++创建蓝图资源
收到一位朋友公司奇怪的需求,遍历场景内的StaticMeshActor,为每一个都创建一个独立的蓝图资源,并且这个蓝图内包含这个StaticMeshActor.  
引用场景并不多,但是我确实不知道如何用代码创建蓝图资源,并且使用代码修改这个蓝图.  
通过万能的google我还是查到了关键函数**FKismetEditorUtilities::CreateBlueprint**  
通过阅读这个Kismet函数，发现实际上创建一个蓝图流程非常复杂繁琐，这里就直接使用这个函数创建。

另外CreateBlueprint返回的是一个UBlueprint类型的指针，是没办法想AActor直接attach actor的，依然需要Kismet函数  **FKismetEditorUtilities::AddActorsToBlueprint**  
最后调用**FAssetRegistryModule::AssetCreated(newBlueprint);**  
细节参考以下代码

```C++
///这个函数将暴露给蓝图使用 参数将由蓝图传递
void UCreateBPTapPage::CreateLevelBPAssets(TSubclassOf<AActor> ActorTypeParam,const TSoftObjectPtr<UWorld> Level,FString ParentPath, FString BPName)
{
	
///--------------------获取蓝图类和生成的蓝图类---------------------
	UClass* BlueprintClass = nullptr;
	UClass* BlueprintGeneratedClass = nullptr;
	IKismetCompilerInterface& KismetCompilerModule = FModuleManager::LoadModuleChecked<IKismetCompilerInterface>("KismetCompiler");
    
	KismetCompilerModule.GetBlueprintTypesForClass(AActor::StaticClass(), BlueprintClass, BlueprintGeneratedClass);
///---------------------------------------------------------------

///获取关卡资源内包含的Actor 并且判断筛选出特定类型
	for(TActorIterator<AActor> ActorItr(Level->GetWorld());ActorItr;++ActorItr )
	{
		if(ActorItr->IsA(ActorTypeParam))
		{
			FString ActorName = ActorItr->GetName();
			FName BPFName = FName(BPName + ActorName);
			FString PkgPath = FPaths::Combine(ParentPath,  BPFName.ToString());


			UPackage* ParentPackage;
			FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");
			FAssetData ExistingAsset = AssetRegistryModule.Get().GetAssetByObjectPath(FName(*PkgPath));

///这里需要判断Asset是否已经存在，经过这一流程虽然在编辑器中已经可以看见，实际上蓝图文件并没有保存在本地磁盘上，所以需要通过AssetRegistryModule判断资源是否已经存在，如果已经存在还试图创建相同名称的UPackage，编辑器将直接闪退。
			if (ExistingAsset.IsValid())
			{
				UE_LOG(LogTemp,Error,TEXT("%s file had exist"),*PkgPath);
				continue;
			}
			
			ParentPackage = CreatePackage(*PkgPath);
			ParentPackage->FullyLoad();
			UBlueprint* newBlueprint = FKismetEditorUtilities::CreateBlueprint(  AActor::StaticClass(), ParentPackage, BPFName, BPTYPE_Normal, BlueprintClass, BlueprintGeneratedClass);
			TArray<AActor*> tempList{*ActorItr};
			FKismetEditorUtilities::FAddActorsToBlueprintParams parms;
			parms.bReplaceActors = false;
			FKismetEditorUtilities::AddActorsToBlueprint(newBlueprint,tempList,parms);
			FAssetRegistryModule::AssetCreated(newBlueprint);
			
		}
	}
}
```