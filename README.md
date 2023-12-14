Small and experimental Amazon S3-compatible object storage "client" in a single file.

Inspired by
- https://gist.github.com/chrismccord/37862f1f8b1f5148644b75d20d1cb073 (single file, easy)
- https://github.com/aws-beam/aws-elixir (clients are structs, xmerl)
- https://github.com/ex-aws/ex_aws_s3 (streaming uploads and downloads)

Verified to work with Amazon S3, MinIO.

TODO: Wasabi, Backblaze B2, Cloudflare R2, DigitalOcean, and Scaleway.

#### Example using [MinIO](https://github.com/minio/minio) and [Finch](https://github.com/sneako/finch)

```console
$ docker run -d --rm -p 9000:9000 -p 9001:9001 --name minio minio/minio server /data --console-address ":9001"
$ docker exec minio mc md testbucket
$ iex
```

```elixir
iex> Mix.install([:finch, {:s3, github: "ruslandoga/s3"}])
iex> Finch.start_link(name: MinIO.Finch)
iex> config = fn options -> 
 Keyword.merge(
    [
      access_key_id: "minioadmin",
      secret_access_key: "minioadmin",
      url: URI.parse("http://localhost:9000"),
      region: "us-east-1"
    ],
    options
  )
end

# PutObject
iex> {uri, headers, body} = S3.build(
  config.(
    method: :put,
    headers: [{"content-type", "application/octet-stream"}],
    path: "/testbucket/my-bytes",
    body: <<0::1000000-bytes>>
  )
)
iex> req = Finch.build(:put, uri, headers, body)
iex> Finch.request!(req, MinIO.Finch).status
200

# HeadObject
iex> {uri, headers, body} = S3.build(
  config.(method: :head, path: "/testbucket/my-bytes")
)
iex> req = Finch.build(:head, uri, headers, body)
iex> Map.new(Finch.request!(req, MinIO.Finch).headers)
%{
  "content-length" => "1000000",
  "content-type" => "application/octet-stream",
  "etag" => "\"879f4bba57ed37c9ec5e5aedf9864698\""
  # etc.
}

# stream GetObject
iex> {uri, headers, body} = S3.build(
  config.(method: :get, path: "/testbucket/my-bytes")
)
iex> req = Finch.build(:get, uri, headers, body)
iex> stream = fn
  {:data, data}, _ -> IO.inspect(byte_size(data), label: "bytes received")
  _, _ -> :ok
end
iex> Finch.stream(req, MinIO.Finch, _acc = [], stream)
# bytes received: 147404
# bytes received: 408300
# bytes received: 408300
# bytes received: 35996

# chunked PutObject
# https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-streaming.html
iex> stream = <<0::10000-bytes>> |> Stream.repeatedly() |> Stream.take(100)
iex> {uri, headers, {:stream, stream}} = S3.build(
  config.(
    method: :put,
    headers: [{"content-type", "application/octet-stream"}],
    path: "/testbucket/my-bytestream",
    body: {:stream, stream}
  )
)
iex> req = Finch.build(:put, uri, headers, {:stream, stream})
iex> Finch.request!(req, MinIO.Finch).status
200

# ListObjectsV2
# https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html
iex> {uri, headers, body} = S3.build(
  config.(method: :get, query: %{"list-type" => 2})
)
iex> req = Finch.build(:get, uri, headers, body)
iex> S3.xml(Finch.request!(req, MinIO.Finch).body)
[
  {
    "ListBucketResult",
    [
      {"Name", "testbucket"},
      {"Prefix", []},
      {"KeyCount", "2"},
      {"MaxKeys", "1000"},
      {"IsTruncated", "false"},
      {
        "Contents",
        [
          {"Key", "my-bytes"},
          {"LastModified", "2023-12-14T08:54:40.085Z"},
          {"ETag", "\"879f4bba57ed37c9ec5e5aedf9864698\""},
          {"Size", "1000000"},
          {"StorageClass", "STANDARD"}
        ]
      },
      {
        "Contents",
        [
          {"Key", "my-bytestream"},
          {"LastModified", "2023-12-14T08:54:40.042Z"},
          {"ETag", "\"879f4bba57ed37c9ec5e5aedf9864698\""},
          {"Size", "1000000"},
          {"StorageClass", "STANDARD"}
        ]
      }
    ]
  }
]
```

```console
$ docker stop minio
```

TODO:
- [Signed Upload Form](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-UsingHTTPPOST.html)
- [Signed URL](https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html)
- [DeleteObject](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)
- [DeleteObjects](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjects.html)
