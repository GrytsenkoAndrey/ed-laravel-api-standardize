# ed-laravel-api-standardize

Standardizing success and error response is crucial for an API. Since, API are often consumed by other systems. Laravel make it easier with Responsable Interface, which in short convert object into http response instance. All we have to do is, impelement interface and define toResponse().

We can begin by creating class for success response. It will look as following:

```
<?php

namespace App\Http\Responses;

use Illuminate\Contracts\Support\Responsable;
use Illuminate\Http\Response;

class ApiSuccessResponse implements Responsable
{
    public function __construct(
        private mixed $data,
        private array $metaData,
        private int $statusCode = Response::HTTP_CREATED,
        private array $headers = [],
        private int $options = 0
    ) {
    }

    public function toResponse($request)
    {
        return response()->json(
            [
                'data' => $this->data,
                'metaData' => $this->metaData
            ],
            $this->statusCode,
            $this->headers,
            $this->options
        );
    }
}
```

Similar to above class, our Error class will look as following:

```
<?php

namespace App\Http\Responses;

use Exception;
use Illuminate\Contracts\Support\Responsable;
use Illuminate\Http\Response;

class ApiErrorResponse implements Responsable
{
    public function __construct(
        private ?Exception $exception,
        private string $message,
        private int $statusCode = Response::HTTP_BAD_REQUEST,
        private array $headers = [],
        private int $options = 0
    ) {
    }

    public function toResponse($request)
    {
        $response = ['message' => $this->message];

        if (!empty($this->exception) && config('app.debug')) {
            $response['debug'] = [
                'message' => $this->exception->getMessage(),
                'file' => $this->exception->getFile(),
                'line' => $this->exception->getLine(),
                'trace' => $this->exception->getTrace()
            ];
        }

        return response()->json(
            $response,
            $this->statusCode,
            $this->headers,
            $this->options
        );
    }
}
```

And then we can use above classes in our controller to standardize success and error response.

```
/**
     * Update the product.
     * @param Request
     * @param String
     */
    public function update(Request $request, string $id)
    {
        if (!$request->user()->tokenCan('update')) {
            return new ApiErrorResponse(
                new AuthorizationException(),
                'User not authorized to perform operation',
                Response::HTTP_FORBIDDEN
            );
        }

        $data = $request->validate([
            'name' => 'required|string',
            'color' => 'required|string',
            'description' => 'required|string',
            'price' => 'required|decimal:2'
        ]);

        $product = Product::find($id);

        if (!empty($product)) {
            $product->update($data);
            return new ApiSuccessResponse(
                $product,
                ['message' => 'Product updated Successfully']
            );
        }

        return new ApiErrorResponse(
            new Exception(),
            "Error updating Product",
            Response::HTTP_NOT_FOUND,
        );
    }
```

In above function, we are updating an existing product. We begin with checking whether user is authorized to perform update operation with token passed along with the request. If user is not authorized, then we use our newly created ApiErrorResponse class to create Error response. It will look like:

```
{
    "message": "User not authorized to perform operation"
}
```

If we have _APP_DEBUG=true_ in **.env**, then it will show additional debug key with more details.

Next, we do the validation of input request. If we have any Illuminate\Validation\ValidationException error, then system return in following format which is somewhat similar to our format.

```
{
    "message": "The color field is required.",
    "errors": {
        "color": [
            "The color field is required."
        ]
    }
}
```

Next, If we found our Product, we will use ApiSuccessResponse class to respond with success message.

```
{
    "data": {
        "id": 102,
        "name": "user 102",
        "color": "purple",
        "description": "i updated the user",
        "price": 1072.22,
        "created_at": "2023-09-26 04-31-47",
        "updated_at": "2023-09-28 03-20-12"
    },
    "metaData": {
        "message": "Product updated Successfully"
    }
}
```

Next, if we didn’t found any product related to $id passed. We return error response which will look like:

```
{
    "message": "Error updating Product"
}
```

We can make more improvement in API response object like use Resource to format created_at into createdAt according to REST API standards.
