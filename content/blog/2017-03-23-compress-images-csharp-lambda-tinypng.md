---
date: 2017-03-23T00:00:00Z
description: |
  Get started with C# and .NET Core by creating a Lambda function which compresses images uploaded to S3 using TinyPNG.
tags:
- .net core
- aws
- aws lambda
- aws s3
title: Compress images with C#, .NET Core, AWS Lambda and TinyPNG
url: /blog/compress-images-csharp-lambda-tinypng/
---

In this blog post we will look at how you can create a simple [AWS Lambda](https://aws.amazon.com/lambda/) function in C# (and .NET Core) which will compress images uploaded to an S3 bucket using the [TinyPNG](https://tinypng.com/) API. The Lambda function will be configured to automatically be triggered whenever a new image is uploaded to the S3 bucket.

I am using Visual Studio 2017, so ensure you have downloaded and installed the [Preview of the AWS Toolkit for Visual Studio 2017](https://aws.amazon.com/blogs/developer/preview-of-the-aws-toolkit-for-visual-studio-2017/).

## Sign up for TinyPNG

I will be using TinyPNG to compress the images from the Lambda function, so if you want to follow along, then please head over to the [TinyPNG Developer website](https://tinypng.com/developers) and sign up for an API Key.

Save the API Key you received after signing up, as you will require it later in this blog post to pass along when calling the TinyPNG API.

## Creating the Lambda function

To get started create a new Lambda project in Visual Studio:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/new-lambda-project.png)

For the Lambda Blueprint you can select an empty function:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/select-lambda-blueprint.png)

Next up install the NuGet packages we will require:

```csharp
Install-Package Amazon.Lambda.S3Events
Install-Package TinyPNG
```

The full code for the function is as follows:

```csharp
public class Function
{
    private readonly string[] _supportedImageTypes = new string[] {".png", ".jpg", ".jpeg"};
    private readonly AmazonS3Client _s3Client;

    public Function()
    {
        _s3Client = new AmazonS3Client();
    }

    public async Task FunctionHandler(S3Event s3Event, ILambdaContext context)
    {
        foreach (var record in s3Event.Records)
        {
            if (!_supportedImageTypes.Contains(Path.GetExtension(record.S3.Object.Key).ToLower()))
            {
                Console.WriteLine(
                    $"Object {record.S3.Bucket.Name}:{record.S3.Object.Key} is not a supported image type");
                continue;
            }

            Console.WriteLine(
                $"Determining whether image {record.S3.Bucket.Name}:{record.S3.Object.Key} has been compressed");

            // Get the existing tag set
            var taggingResponse = await _s3Client.GetObjectTaggingAsync(new GetObjectTaggingRequest
            {
                BucketName = record.S3.Bucket.Name,
                Key = record.S3.Object.Key
            });

            if (taggingResponse.Tagging.Any(tag => tag.Key == "Compressed" && tag.Value == "true"))
            {
                Console.WriteLine(
                    $"Image {record.S3.Bucket.Name}:{record.S3.Object.Key} has already been compressed");
                continue;
            }

            // Get the existing image
            using (var objectResponse = await _s3Client.GetObjectAsync(record.S3.Bucket.Name, record.S3.Object.Key))
            using (Stream responseStream = objectResponse.ResponseStream)
            {
                Console.WriteLine($"Compressing image {record.S3.Bucket.Name}:{record.S3.Object.Key}");

                // Use TinyPNG to compress the image
                TinyPngClient tinyPngClient = new TinyPngClient(Environment.GetEnvironmentVariable("TinyPNG_API_Key"));
                var compressResponse = await tinyPngClient.Compress(responseStream);
                var downloadResponse = await tinyPngClient.Download(compressResponse);
                    
                // Upload the compressed image back to S3
                using (var compressedStream = await downloadResponse.GetImageStreamData())
                {
                    Console.WriteLine($"Uploading compressed image {record.S3.Bucket.Name}:{record.S3.Object.Key}");

                    await _s3Client.PutObjectAsync(new PutObjectRequest
                    {
                        BucketName = record.S3.Bucket.Name,
                        Key = record.S3.Object.Key,
                        InputStream = compressedStream,
                        TagSet = new List<Tag>
                        {
                            new Tag
                            {
                                Key = "Compressed",
                                Value = "true"
                            }
                        }
                    });
                }
            }
        }
    }
}
```

Let's walk through the logic for the code above:

1. The function is declared to take an `S3Event` as input parameter (along with an instance of `ILambdaContext`). The `S3Event` will contain information about the S3 event which triggered the function, such as uploading of a new file to a bucket. The function will process each of the records in the S3 event notification.
2. If the S3 Object's file extension is not in the list if valid image extensions we are interested in, then the object will be skipped.
3. If the S3 Object is a valid image, the tags of the object will be checked for the presence of a tag named `Compressed` with a value of `true`. This will be our indicator that we have processed and compressed a particular image already. If it has been compressed, it will be skipped.
4. At this point we have a valid, uncompressed image. So we read it from S3 into a stream.
5. An new instance of `TinyPngClient` is created and we read the value for the TinyPNG API key from an environment variable named "TinyPNG_API_Key". (This will be configured later when deploying the function.)
6. The stream is passed to the instance of `TinyPngClient` to upload and compress the image using the TinyPNG API.
7. Finally the compressed image is downloaded from TinyPNG, and then uploaded to the same S3 bucket with the same name as before (i.e. replacing the existing image). This time however we will add a tag named `Compressed` with a value of `true` to the image to ensure that it does not get processed a second time around.

## Deploying the Lambda

To deploy the Lambda function to AWS, right-click on your project in the Solution Explorer window in VS 2017, and select the **Publish to AWS Lambda...** option. This will open the **Upload Lambda Function** dialog box. You can complete the information by supplying a **Function Name**:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/upload-lambda.png)

Click on the **Next** button and complete the **Advanced Lambda Settings**. In the **Environment** section add a new Variable named `TinyPNG_API_Key` with the value of the API key you received from TinyPNG:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/advanced-lambda-settings.png)

Click on the **Upload** button.

## Testing the function

Once the Lambda function has been uploaded, you can test it. I have an existing S3 bucket with some images which I uploaded before:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/images-before-compression.png)

If I browse the bucket in Visual Studio I can right click on the images and select the **Invoke Lambda Function...** option:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/invoke-lambda-menu.png)

This will open up the **Invoke Lambda Function** dialog window:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/invoke-lambda-dialog.png)

Ensure that you have selected the new Lambda function and click on OK. Give it a while, and when you refresh your bucket, you should notice that the images are now considerable smaller:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/images-after-compression.png)

## Automatically invoking the function

You can also configure the function to automatically invoke every time a new image is uploaded to the S3 Bucket. To do this you can open the Lambda function's settings inside Visual Studio and go to the **Event Sources** tab:

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/lambda-event-sources.png)

Click the **Add** button and in the **Add Event Source** dialog, select **Amazon S3** as the event source, and select the S3 bucket you want to monitor. Click OK to add the event source.

![](/assets/images/2017-03-23-compress-images-csharp-lambda-tinypng/add-event-source-dialog.png)

Now, every time you upload a new image to that particular bucket the Lambda function will automatically be triggered and the image will be compressed.

## Conclusion

In this blog post we developed a simple AWS Lambda function using C# and .NET Core. The function monitors an AWS S3 Bucket, and every time a new image is added to the bucket the image is automatically compressed using the TinyPNG API.

Source code for this blog post can be found at [https://github.com/jerriepelser-blog/LambdaImageCompressor](https://github.com/jerriepelser-blog/LambdaImageCompressor)