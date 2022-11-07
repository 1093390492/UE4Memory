## UE4 Pak的打包与挂载 （修订版）

基本的Pak资源打包与加载流程

1. CookContent 烘焙uasset文件
2. UnrealPak 打包Pak文件
3. FPakFile，FPakPlatformFile 从Pak文件中遍历文件 加载特定类型的UObject
4. SpawnActor 在世界中创建物体
   
---
### CookContent 烘焙uasset文件
可是直接使用菜单栏File->Cook Content for Windows将项目内所有资源全部打包，然后在项目目录下[ProjectName]->Saved->Cooked->WindowsNoEditor->ProjectName->Content找到对应的Cook好的资源。  
也可以使用命令行打包  (`https://docs.unrealengine.com/4.27/zh-CN/SharingAndReleasing/Deployment/Cooking/`)  
例：
`UE4Editor.exe <GameName or uproject> -run=cook -targetplatform=<Plat1>+<Plat2> [-cookonthefly] [-iterate] [-map=<Map1>+<Map2>]`  
或  
`UE4Editor-Cmd.exe <GameName> -run=cook -targetplatform=<Plat1>+<Plat2> [-cookonthefly] [-iterate] [-map=<Map1>+<Map2>] `  
`Example UE4Editor-Cmd.exe C:\SoftwareInstallation\EpicGames\UE_4.26\Engine\Binaries\Win64\UE4Editor-Cmd.exe
 "C:\Users\Alice\Documents\Unreal Projects\DesignPatterns\DesignPatterns.uproject"
 -run=Cook  -TargetPlatform=WindowsNoEditor -unversioned  -stdout -CrashForUAT -unattended -NoLogTimes  -UTF8Output`

该commandlet必须通过`-run=cook`指定，还必须指定要烘焙的平台。该命令会为指定的平台生成数据， 并将数据保存在以下位置：  
`<Game>/Saved/Sandboxes/Cooked-<Platform>`  
  
|  选项   | 说明  |
|  ----  | ----  |
| `-targetplatform=<Plat1>+<Plat2>`  | `指定要烘焙的平台。可用平台列表包含WindowsNoEditor、WindowsServer、LinuxServer、PS4, XboxOne、IOS和Android。` |
| `-iterate`  | `指定烘焙器仅烘焙过时项目。如果不指定该选项，则沙箱目录将被删除，所有内容将重新烘焙。` |
| `-Map=<Map1>+<Map2>+...`  | `指定要构建的贴图。` |
| `-cookonthefly`  | `指定以服务器模式启动烘焙器。这样将启动服务器，服务器将等待游戏连接，然后根据需要提供烘焙的数据。使用该选项时，游戏需要在其命令行上指定-filehostip=<Server IP>以便能够连接服务器。` |
| `-MapIniSection=<ini file section>`  | `指定ini文件中包含贴图名称的分段。烘焙器将烘焙指定分段中指定的所有贴图。` |
| `-UnVersioned`  | `保存所有烘焙的数据包，不含版本。然后这些数据包在加载时会被假定为最新版本。` |
| `-CookAll`  | `烘焙所有内容。` |
| `-Compressed`  | `告知烘焙器压缩烘焙过的数据包。` |  


### UnrealPak 打包Pak文件
Cook完成后找到 UnrealPak.exe文件，位于Engine\Binaries\Win64目录下，命令行启动。
常用命令  

|  命令   | 说明  |
|  ----  | ----  |
|UnrealPak Test.pak –create=D:\MyUE4Projects\SomeFolder\ |将指定文件夹下的文件，打包到指定目录下的Pak文件|
|UnrealPak Test.pak –create=D:\MyUE4Projects\SomeFolder\ -compress |创建压缩后的Pak|
|UnrealPak Test.pak –create=D:\MyUE4Projects\SomeFolder\ -aes=77777777777777777777777777777777 -encryptindex |给Pak加秘钥|
|UnrealPak Test.pak -list |查看列出Pak文件中的信息|
|UnrealPak Test.pak –extract=E:\ExtractedFolder\ |解压Pak文件到指定目录|  

### 从Pak加载UObject && SpawnActor 在世界中创建物体
1. 创建FPakPlatformFile并初始化
2. 加载Pak前使用FPlatformFileManager::Get().SetPlatformFile切换到Pak平台加载Pak后，再切换回原有的平台
3. 创建FPakFile，根据原有MountPoint拼出新的MountPoint并应用E:/UE4Proj/DBJTest/Saved/Cooked/WindowsNoEditor/DBJTest/Content/DLC/转换为../../../DBJTest/Content/DLC/
4. PakPlatform->Mount
5. 使用FindFilesAtPath从FPakFile中获取Pak文件中的文件列表
6. 根据文件名拼出新的文件名路径，使用StaticLoadObject加载工程下的资源路径标准：/Game/DLC/Cube

### 要注意的点
相比4.27之前的版本 
1. FPakFile由原来的TSharedPtr智能指针改为了TRefCountPtr指针
2. 实际上在切换文件平台至Pak平台加载pak文件之后，并不需要立即切换回默认平台（物理平台），因为文件平台管理系统的责任链机制，在当前平台查不到所需资源后，会再到下层。也就是LowerLevel所指向的平台，在IPlatformFile的函数Initialize，第一个参数就是LowerLevel，也就是说，pak平台无法加载的资源，会到默认平台（物理平台）再次查询。
3. 关闭的时候需要将平台切换回默认平台，否则退出的时候会崩溃，原因暂时未知。
4. 在PIE模式下 MountPoint需要使用绝对路径 打包后需要使用相对路径，原因猜测为底层寻址逻辑相关。
5. 之后就按照常规模式，使用MountPoint设定的路径加载资源
#### 个人修订版 示例代码
```C++
void UPakSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);

	OriPlatform = &FPlatformFileManager::Get().GetPlatformFile();
	
	if(!FPlatformFileManager::Get().FindPlatformFile(FPakPlatformFile::GetTypeName()))
	{
		PakPlatform = MakeShareable(new FPakPlatformFile());

		//该初始化函数实际上就是指认了下层责任平台
		PakPlatform->Initialize(&FPlatformFileManager::Get().GetPlatformFile(), TEXT(""));
		//直接在初始化的时候 切换到pak 后面直接加载pak 不再需要切换
		FPlatformFileManager::Get().SetPlatformFile(*PakPlatform);
	}
}

void UPakSubsystem::Deinitialize()
{
	Super::Deinitialize();

	//退出时在切换回默认平台 否则会闪退5.02甚至错误都不报。
	FPlatformFileManager::Get().SetPlatformFile(*OriPlatform);
}

bool UPakSubsystem::MountPak(FString PakFullPath)
{
	if(!PakFullPath.EndsWith(".pak"))
	{
		UE_LOG(LogTemp,Error,TEXT("PakFullPath '%s' not End with .pak"),*PakFullPath);
		return false;
	}
	
	
	if (!PakPlatform->FileExists(*PakFullPath))
	{
		UE_LOG(LogTemp,Error,TEXT("PakFullPath '%s' not Exists"),*PakFullPath);
		return false;
	}

	if(!FPlatformFileManager::Get().FindPlatformFile(FPakPlatformFile::GetTypeName()))
	{
		UE_LOG(LogTemp,Error,TEXT("FPlatformFileManager  not Exists FPakPlatformFile"));
		return false;
	}

	TRefCountPtr<FPakFile> PakFile = new FPakFile(PakPlatform.Get(),*PakFullPath,false);
	

	if(!PakFile)
	{
		UE_LOG(LogTemp,Error,TEXT("Create PakFile Error"));
		return false;
	}

	TArray<FString> MountedPakFilenames;
	PakPlatform->GetMountedPakFilenames(MountedPakFilenames);

	

	
	if( MountedPakFilenames.Find(FPaths::GetBaseFilename(PakFullPath)) > -1)
	{
		UE_LOG(LogTemp,Warning,TEXT("Pak had Mounted"));
		return false;
	}

//在PIE模式下 MountPoint需要使用绝对路径 打包后需要使用相对路径
#if !WITH_EDITOR
	FString PakMountPoint = PakFile->GetMountPoint();
	UE_LOG(LogTemp,Log,TEXT("PakMountPoint %s"),*PakMountPoint);
	int Pos = PakMountPoint.Find("Content/");
	FString NewMountPoint = PakMountPoint.RightChop(Pos);
	NewMountPoint =  FPaths::ProjectDir() + NewMountPoint;
	PakFile->SetMountPoint(*NewMountPoint);
#endif
	if(PakPlatform->Mount(*PakFullPath,1,*PakFile->GetMountPoint()))
	{
		
		UE_LOG(LogTemp,Log,TEXT("Pak Mounted success %s"),*PakFile->GetMountPoint());

		TArray<FString> FoundFilenames;
		PakFile->FindFilesAtPath(FoundFilenames, *PakFile->GetMountPoint(), true, false, false);
		for (FString& Filename : FoundFilenames)
		{
			UE_LOG(LogTemp,Log,TEXT("FoundFilenames %s"),*Filename);
		}
		return  true;
	}
	else
	{
		
		UE_LOG(LogTemp,Error,TEXT("Pak Mounted fail %s"),*PakFile->GetMountPoint());
		return  false;
	}

	
}
```

#### 旧的 示例代码
```C++
void AMyPlayerController::BeginPlay()
{
	Super::BeginPlay();

    //记录原本的文件平台 在pak读取完毕之后需要切换回来
	OldPlatform = &FPlatformFileManager::Get().GetPlatformFile();
	PakPlatform = MakeShareable(new FPakPlatformFile());
	PakPlatform->Initialize(&FPlatformFileManager::Get().GetPlatformFile(), TEXT(""));
}


void AMyPlayerController::TestLoadPak(FString InPakFullPath)
{
	FPlatformFileManager::Get().SetPlatformFile(*PakPlatform.Get());

	FString PakFileFullPath = InPakFullPath;

	TSharedPtr<FPakFile> TmpPak = MakeShareable(new FPakFile(PakPlatform.Get(), *PakFileFullPath, false));
	FString PakMountPoint = TmpPak->GetMountPoint(); //原本打包pak时候的绝对路径 需要截断转换为项目的相对路径
	int32 Pos = PakMountPoint.Find("Content/");

	FString NewMountPoint = PakMountPoint.RightChop(Pos);
	NewMountPoint = FPaths::ProjectDir() + NewMountPoint;//获得的相对路径 相当于硬盘上的虚拟目录
	//FString NewMountPoint = "../../../DBJTest/Content/DLC/";


	TmpPak->SetMountPoint(*NewMountPoint);

	if (PakPlatform->Mount(*PakFileFullPath, 1, *NewMountPoint))
	{
		TArray<FString> FoundFilenames;
		TmpPak->FindFilesAtPath(FoundFilenames, *TmpPak->GetMountPoint(), true, false, false);

		if (FoundFilenames.Num() > 0)
		{
			for (FString& Filename : FoundFilenames)
			{
				if (Filename.EndsWith(TEXT(".uasset")))
				{
					FString NewFileName = Filename;
					NewFileName.RemoveFromEnd(TEXT(".uasset"));
					int32 Pos = NewFileName.Find("/Content/");
					NewFileName = NewFileName.RightChop(Pos + 8);
					NewFileName = "/Game" + NewFileName;
                    //上述操作将虚拟目录的地址转换成UE的资源引用路径


					UObject* LoadedObj = StaticLoadObject(UObject::StaticClass(), NULL, *NewFileName);

					UStaticMesh* SM = Cast<UStaticMesh>(LoadedObj);
					if (SM)
					{
						AStaticMeshActor* MeshActor = GetWorld()->SpawnActor<AStaticMeshActor>(AStaticMeshActor::StaticClass(), FVector(0,0,460), FRotator(0,0,0) );
						MeshActor->SetMobility(EComponentMobility::Movable);
						MeshActor->GetStaticMeshComponent()->SetStaticMesh(SM);
						break;
					}
				}
			}
		}
	}


    //切换回原本的平台
	FPlatformFileManager::Get().SetPlatformFile(*OldPlatform);

}

```


