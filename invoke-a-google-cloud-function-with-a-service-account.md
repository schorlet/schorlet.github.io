# Invoke a Google Cloud Function with a Service Account

This article is about the [Service-to-function](https://cloud.google.com/functions/docs/securing/authenticating#service-to-function) section of the Google Cloud documentation.

The context:

- You have deployed a HTTP function which requires authentication.
- You have created a service account that is granted the permission to invoke the function.
- You need to invoke the function from outside GCP.

When you restrict the ability to invoke a function. it can only be invoked by providing authentication credentials in the request.

As a developer, you can use the `gcloud` tool and test the function as follows:

```sh
gcloud auth activate-service-account $SERVICE_ACCOUNT --key-file $KEY_FILE

curl https://${REGION}-${PROJECT_ID}.cloudfunctions.net/${FUNCTION_NAME} \
	-H "Authorization: bearer $(gcloud auth print-identity-token)"
```

Reading the documentation, if you're invoking a function from your application, you'll have to manually generate the proper token:

1. Self-sign a service account JWT with the **target_audience** claim set to the URL of the receiving function.
1. Exchange the self-signed JWT for a Google-signed ID token, which should have the aud claim set to the above URL.
1. Include the ID token in an "Authorization: Bearer **ID_TOKEN**" header in the request to the function.

This can be a [lot of work](https://github.com/salrashid123/google_id_token/blob/master/golang/GoogleIDToken.go), but you can do (almost) the same with a few lines of code:

```go
import (
	"context"
	"fmt"
	"io/ioutil"

	"golang.org/x/oauth2/google"
	"golang.org/x/oauth2/jwt"
)

func invoke(credentials, functionURL string) error {
	data, err := ioutil.ReadFile(credentials)
	if err != nil {
		return err
	}

	conf, err := google.JWTConfigFromJSON(data)
	if err != nil {
		return err
	}
	conf.PrivateClaims = map[string]interface{}{"target_audience": functionURL}
	conf.UseIDToken = true

	client := conf.Client(context.Background())
	resp, err := client.Get(functionURL)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return err
	}
	fmt.Println(string(body))

	return nil
}
```

The http.Client created by the [jwt.Config](https://pkg.go.dev/golang.org/x/oauth2/jwt) will take care of:

- Create and sign a JWT (Json Web Token).
- Exchange the Signed-JWT for a Google ID Token.
- Automatically set the Authorization Header for each request.
- And reuse/renew the ID Token.


