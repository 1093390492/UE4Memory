
## UE4提交自定义渲染指令
以渲染线程抓图并且保存到本地为例

---
### ENQUEUE_RENDER_COMMAND
ENQUEUE_RENDER_COMMAND 表示从游戏线程向渲染线程发送渲染指令
该宏传入一个Lamda函数，用该Lamda创造一个GraphTask任务加入TaskGraph等待执行
``` C++
	ENQUEUE_RENDER_COMMAND(CaptureJobData)([this,TempCaptureComponent = this->CaptureComponent,URL](FRHICommandListImmediate& RHICmdList)
	{
		if(TempCaptureComponent && TempCaptureComponent->TextureTarget)
		{

			//GetRenderTargetResource函数要求在Render线程执行
			FTextureRenderTargetResource* RenderResource = TempCaptureComponent->TextureTarget->GetRenderTargetResource();
			//FTextureRenderTargetResource* RenderResource = TempCaptureComponent->TextureTarget->GameThread_GetRenderTargetResource(); GameThread_GetRenderTargetResource函数会要求游戏线程执行，否则断言无法通过。

			int Width = RenderResource->GetSizeX();
			int Height = RenderResource->GetSizeY();

			TArray<FColor> ImageColorList;
			ImageColorList.AddUninitialized(Width * Height);

			//FTextureRenderTargetResource类成员函数ReadPixels调用FlushRenderingCommands会断言GameThread
			//FlushRenderingCommands只能在GameThread调用，用于等待一帧渲染完毕，这里为了测试
			//实现了自定义的ReadPixels 注释掉FlushRenderingCommands函数的调用
			ReadPixels(RenderResource,ImageColorList);

			FString FullPath = URL;

			//接下来的图像处理以及文件本地保存建议放到独立线程内执行
			TArray<uint8> CompressedBitmap;
			FImageUtils::CompressImageArray(Width,Height,ImageColorList,CompressedBitmap);
			FFileHelper::SaveArrayToFile(CompressedBitmap,*FullPath);
			
		}
		else
		{
			UE_LOG(LogTemp,Error,TEXT("CaptureComponent is NULL or CaptureComponent->TextureTarget is NULL"));
		}
	});

```