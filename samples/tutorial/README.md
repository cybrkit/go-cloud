# Getting Started With The Go Cloud Development Kit

The best way to understand the Go Cloud Development Kit (Go CDK) is to write
some code and use it. Let's build a command line application that uploads files
to blob storage on both AWS and GCP. Blob storage stores binary data under a
string key, and is one of the most frequently used cloud services. We'll call
the tool `upload`.

Before we start with any code, it helps to think about possible designs. We
could very well write a code path for Amazon's Simple Storage Service (S3) and
another code path for Google Cloud Storage (GCS). That would work. However, it
would also be tedious. We would have to learn the semantics of uploading files
to both blob storage services. Even worse, we would have two code paths that
effectively do the same thing, but would have to be maintained separately
(ugh!). It would be much nicer if we could write the upload logic once and reuse
it across providers. That's exactly the kind of
[separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)
that the Go CDK makes possible. So, let's write some code!

When we're done, our command line application will work like this:

```shell
# uploads gopher.png to GCS
$ ./upload -cloud gcp gopher.png

# uploads gopher.png to S3
$ ./upload -cloud aws gopher.png

# uploads gopher.png to Azure
$ ./upload -cloud azure gopher.png
```

## Flags & setup

We start with a skeleton for our program with flags to configure the cloud
provider and the bucket name.

```go
// Command upload saves files to blob storage on GCP, AWS, and Azure.
package main

import (
    "flag"
    "log"
)

func main() {
    // Define our input.
    cloud := flag.String("cloud", "", "Cloud storage to use")
    bucketName := flag.String("bucket", "go-cloud-bucket", "Name of bucket")
    flag.Parse()
    if flag.NArg() != 1 {
        log.Fatal("Failed to provide file to upload")
    }
    file := flag.Arg(0)
}
```

Now that we have a basic skeleton in place, let's start filling in the pieces.
Opening a connection to the bucket will depend on which cloud we're using.
Because we allow the user to specify a cloud, we will write connection logic for
both clouds. This will be the only step that requires cloud-specific APIs.

## GCP authentication & connection

As a first pass, let's write the code to connect to a GCS bucket. Then, we will
write the code to connect to S3 and Azure buckets. In all cases, we are going to
create a pointer to a `blob.Bucket`, the cloud-agnostic blob storage type.

```go
package main

import (
    "context"
    "flag"
    "log"

    "gocloud.dev/blob"
    "gocloud.dev/blob/gcsblob"
    "gocloud.dev/gcp"
)

func main() {
    // flag setup omitted
    // ...

    ctx := context.Background()
    // Open a connection to the bucket.
    var (
        b *blob.Bucket
        err error
    )
    switch *cloud {
    case "gcp":
        b, err = setupGCP(ctx, *bucketName)
    case "aws":
        // setupAWS is added in the next code sample.
        b, err = setupAWS(ctx, *bucketName)
    case "azure":
        // setupAzure is added in the next code sample.
        b, err = setupAzure(ctx, *bucketName)
    default:
        log.Fatalf("Failed to recognize cloud. Want gcp or aws, got: %s", *cloud)
    }
    if err != nil {
        log.Fatalf("Failed to setup bucket: %s", err)
    }
}

// setupGCP creates a connection to Google Cloud Storage (GCS).
func setupGCP(ctx context.Context, bucket string) (*blob.Bucket, error) {
    // DefaultCredentials assumes a user has logged in with gcloud.
    // See here for more information:
    // https://cloud.google.com/docs/authentication/getting-started
    creds, err := gcp.DefaultCredentials(ctx)
    if err != nil {
        return nil, err
    }
    c, err := gcp.NewHTTPClient(gcp.DefaultTransport(), gcp.CredentialsTokenSource(creds))
    if err != nil {
        return nil, err
    }
    return gcsblob.OpenBucket(ctx, c, bucket, nil)
}
```

The cloud-specific setup is the tricky part as each cloud has its own series of
invocations to create a bucket. It's worth thinking about this point for a
moment.

Without the Go CDK, both your setup code and your business logic are platform
dependent. That's a bad thing because business logic has a tendency to grow
increasingly complex over time and if that logic is coupled to a particular
platform, your application will likely become tightly coupled to the plaform as
well.

With the Go CDK, only your setup code is platform dependent. The business logic
makes no demands on a particular platform, which means your application itself
requires less work to move from one cloud to another. In any case, the setup
code in an application grows far less than the business logic does. Anything we
can do to separate the business logic from an underlying platform is well worth
doing.

## AWS and Azure authentication & connection

With GCS setup code out of the way, let's handle the S3 and Azure connection
logic next:

```go
package main

import (
    // ...
    // listing only new import statements

    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/credentials"
    "github.com/aws/aws-sdk-go/aws/session"
    "gocloud.dev/blob/azureblob"
    "gocloud.dev/blob/s3blob"
)

// ... main

// setupAWS creates a connection to Simple Cloud Storage Service (S3).
func setupAWS(ctx context.Context, bucket string) (*blob.Bucket, error) {
    c := &aws.Config{
        // Either hard-code the region or use AWS_REGION.
        Region: aws.String("us-east-2"),
        // credentials.NewEnvCredentials assumes two environment variables are
        // present:
        // 1. AWS_ACCESS_KEY_ID, and
        // 2. AWS_SECRET_ACCESS_KEY.
        Credentials: credentials.NewEnvCredentials(),
    }
    s := session.Must(session.NewSession(c))
    return s3blob.OpenBucket(ctx, s, bucket, nil)
}

// setupAzure creates a connection to Azure Storage Account using shared key
// authorization. It assumes environment variables AZURE_STORAGE_ACCOUNT_NAME
// and AZURE_STORAGE_ACCOUNT_KEY are present.
func setupAzure(ctx context.Context, bucket string) (*blob.Bucket, error) {
    accountName := os.Getenv("AZURE_STORAGE_ACCOUNT_NAME")
    accountKey := os.Getenv("AZURE_STORAGE_ACCOUNT_KEY")
    serviceURL, err := azureblob.ServiceURLFromAccountKey(accountName, accountKey)
    if err != nil {
        return nil, err
    }
    return azureblob.OpenBucket(ctx, serviceURL, bucket, nil)
}
```

The important point here is that in spite of the differences in setup for GCS
and S3/Azure, the result is the same: we have a pointer to a `blob.Bucket`.
Provided we go through the setup, once we have a `blob.Bucket` pointer, we may
design our system around that type and prevent any cloud specific concerns from
leaking throughout our application code. In other words, by using `blob.Bucket`
we avoid being tightly coupled to one cloud provider.

With the setup done, we're ready to use the bucket connection. Note, as a design
principle of the Go CDK, `blob.Bucket` does not support creating a bucket and
instead focuses solely on interacting with it. This separates the concerns of
provisioning infrastructure and using infrastructure.

## Reading the file

We need to read our file into a slice of bytes before uploading it. The process
is the usual one:

```go
package main

import (
    // previous imports omitted
    "os"
    "io/ioutil"
)

func main() {
    // ... previous code omitted

    // Prepare the file for upload.
    data, err := ioutil.ReadFile(file)
    if err != nil {
        log.Fatalf("Failed to read file: %s", err)
    }
}
```

## Writing the file to the bucket

Now, we have `data`, our file in a slice of bytes. Let's get to the fun part and
write those bytes to the bucket!

```go
package main

// No new imports.

func main() {
    // ...

    w, err := b.NewWriter(ctx, file, nil)
    if err != nil {
        log.Fatalf("Failed to obtain writer: %s", err)
    }
    _, err = w.Write(data)
    if err != nil {
        log.Fatalf("Failed to write to bucket: %s", err)
    }
    if err := w.Close(); err != nil {
        log.Fatalf("Failed to close: %s", err)
    }
}
```

First, we create a writer based on the bucket connection. In addition to a
`context.Context` type, the method takes the key under which the data will be
stored and the mime type of the data.

The call to `NewWriter` creates a `blob.Writer`, which implements `io.Writer`.
With the writer, we call `Write` passing in the data. In response, we get the
number of bytes written and any error that might have occurred.

Finally, we close the writer with `Close` and check the error.

Alternatively, we could have used the shortcut `b.WriteAll(ctx, file, data,
nil)`.

## Uploading an image

That's it! Let's try it out. As setup, we will need to create an
[S3 bucket][s3-bucket], a [GCS bucket][gcs-bucket], and an
[Azure container][azure-container]. In the code above, I called that bucket
`go-cloud-bucket`, but you can change that to match whatever your bucket is
called. Of course, you are free to try the code on any subset of Cloud
providers.

*   For GCP, you will need to login with
    [gcloud](https://cloud.google.com/sdk/install). If you do not want to
    install `gcloud`, see
    [here](https://cloud.google.com/docs/authentication/production) for
    alternatives.
*   For AWS, you will need an access key ID and a secret access key. See
    [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#Using_CreateAccessKey)
    for details.
*   For Azure, you will need to add your storage account name and key as
    environment variables (`AZURE_STORAGE_ACCOUNT_NAME` and
    `AZURE_STORAGE_ACCOUNT_KEY`, respectively). See
    [here](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal)
    for details.

With our buckets created and our credentials set up, we'll build the program
first:

```shell
$ go build -o upload
```

Then, we will send `gopher.png` (in the same directory as this README) to GCS:

```shell
$ ./upload -cloud gcp gopher.png
```

Then, we send that same gopher to S3:

```shell
$ ./upload -cloud aws gopher.png
```

Finally, we send that same gopher to Azure:

```shell
$ ./upload -cloud azure gopher.png
```

If we check the buckets, we should see our gopher in each of them! We're done!

## Separating setup code from application code

Now that we have a working application, let's take one extra step to further
separate setup code from application code. We will move all setup concerns that
use any specific cloud-provider SDK out into a `setup.go` file and replace our
`switch` statement with a single function:

```go
// main.go

func main() {
    // ...

    cxt := context.Background()
    // Open a connection to the bucket.
    b, err := setupBucket(ctx, *cloud, *bucketName)
    if err != nil {
        log.Fatalf("Failed to setup bucket: %s", err
    }

    // Prepare the file for upload.
    // ...
}
```

After moving the `setupGCP`, `setupAWS`, and `setupAzure` functions into
`setup.go`, we add our new `setupBucket` function next:

```go
// setupBucket creates a connection to a particular cloud provider's blob storage.
func setupBucket(ctx context.Context, cloud, bucket string) (*blob.Bucket, error) {
    switch cloud {
    case "aws":
        return setupAWS(ctx, bucket)
    case "gcp":
        return setupGCP(ctx, bucket)
    case "azure":
        return setupAzure(ctx, bucket)
    default:
        return nil, fmt.Errorf("invalid cloud provider: %s", cloud)
    }
}
```

Now, our application code consists of two files: `main.go` with the application
logic and `setup.go` with the cloud-provider specific details. A benefit of this
separation is that changing any part of blob storage is a matter of changing
only the internals of `setupBucket`. The application logic does not need to be
changed.

## Wrapping Up

In conclusion, we have a program that can seamlessly switch between multiple
Cloud storage providers using just one code path.

I hope this example demonstrates how having one type for multiple clouds is a
huge win for simplicity and maintainability. By writing an application using a
generic interface like `*blob.Bucket`, we retain the option of using
infrastructure in whichever cloud that best fits our needs all without having to
worry about a rewrite.

[s3-bucket]: https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html
[gcs-bucket]: https://cloud.google.com/storage/docs/creating-buckets
[azure-container]: https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction
