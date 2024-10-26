---
layout: post
title: Setting up mocks for Azure Functions Http Triggers
---

Setting up mocks for new SDKs/APIs can be one of the most challenging parts of testing.  Here I'm going to quickly document, via code, how one could set up your xUnit test with mocks for HttpRequestData, HttpResponseData, and DurableTaskClient.

**Code:**

- C# Azure Functions v4 (dotnet-isolated)
- .Net 8.0

Our sample HttpTrigger simply catches an exception and returns an appropriate response:

```c#

using System.Net;
using System.Text.Json;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.Logging;

namespace DemoProjects
{
    public class HttpResponseTrigger
    {
        private readonly ILogger<HttpResponseTrigger> _logger;

        public HttpResponseTrigger(ILogger<HttpResponseTrigger> logger)
        {
            _logger = logger;
        }

        [Function("HttpResponseTrigger")]
        public async Task<HttpResponseData> RunAsync(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get")] HttpRequestData req,
            [DurableClient] DurableTaskClient starter
        )
        {
            // Handling exceptions in Azure functions is critical to retries working correctly.
            try
            {
                var input = await req.ReadAsStringAsync();
                DemoDto dto = JsonSerializer.Deserialize<DemoDto>(input);
                // Do work: put message on queue, start orchestration
            }
            catch (JsonException)
            {
                return req.CreateResponse(HttpStatusCode.UnprocessableEntity);
            }
            catch (Exception)
            {
                return req.CreateResponse(HttpStatusCode.InternalServerError);
            }

            return req.CreateResponse(HttpStatusCode.OK);
        }
    }
}

```

That's simple enough.  Here is the code for a test:

```c#
using System;
using System.Net;
using System.Text;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.DurableTask.Client;
using Microsoft.Extensions.Logging;
using Moq;
using Xunit;

namespace DemoProjects
{
    public class HttpResponseTriggerTests
    {
        private readonly Mock<ILogger<HttpResponseTrigger>> _loggerMock;

        private readonly Mock<HttpRequestData> _requestMock;

        private readonly Mock<HttpResponseData> _responseMock;

        private readonly Mock<DurableTaskClient> _starterMock;

        private readonly HttpResponseTrigger _SUT_httpResponseTrigger;

        public HttpResponseTriggerTests()
        {
            _loggerMock = new Mock<ILogger<HttpResponseTrigger>>();
            _SUT_httpResponseTrigger = new HttpResponseTrigger(_loggerMock.Object);
            Mock<FunctionContext> functionContextMock = new Mock<FunctionContext>();
            _requestMock = new Mock<HttpRequestData>(functionContextMock.Object);
            _responseMock = new Mock<HttpResponseData>(functionContextMock.Object);
            _starterMock = new Mock<DurableTaskClient>("HttpResponseTrigger");

            _requestMock.Setup(r => r.CreateResponse()).Returns(_responseMock.Object);
            _responseMock.SetupProperty(r => r.StatusCode);
        }

        [Fact]
        public async Task HttpResponse_ShouldReturnUnprocessableEntity_OnDeserializationException()
        {
            var content = "invalid content";
            var stream = new MemoryStream(Encoding.UTF8.GetBytes(content));
            _requestMock.Setup(r => r.Body).Returns(stream);

            var result = await _SUT_httpResponseTrigger.RunAsync(_requestMock.Object, _starterMock.Object);

            Assert.Equal(HttpStatusCode.UnprocessableEntity, result.StatusCode);
        }
    }
}
```

You may want to do some setup at a higher level, but this concisely demonstrates how to configure your mocks so that you can start writing tests for those HttpTrigger Functions.  The full project is in [Github](https://github.com/clrockwell/test-azure-function-http-trigger).

