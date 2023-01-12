# Axios Interceptor with TS



## Import from Axios

```typescript
import {
  AxiosError,
  AxiosInstance,
  AxiosRequestConfig,
  AxiosResponse,
} from "axios";
```

타입스크립트에서 각 interceptor의 타입을 명시하기 위해 위의 네 가지를 import 해줍니다.



## Prepare



### Logging

```typescript
const logOnDev = (message: string) => {
  if (import.meta.env.MODE === "development") {
    console.log(message);
  }
};
```

개발 서버 혹은 로컬에서 로그를 남겨 트래킹 하기 위한 코드입니다.



### Loading Handling

```typescript
const onLoading = async (type: string) => {
  const baseStore = useBaseStore();
  const { startLoading, stopLoading, cancelLoading } = baseStore;
  switch (type) {
    case "request":
      startLoading();
      break;
    case "response":
      stopLoading();
      break;
    case "error":
      cancelLoading();
      break;
    default:
      break;
  }

  return Promise.resolve();
};
```

화면에서 로딩 처리를 하기 위해 `store`에 접근하여 상태를 변경하기 위한 코드입니다.



### Error Handling

```typescript
const onError = async (message: string) => {
  const baseStore = useBaseStore();
  const { setAlertMessage } = baseStore;
  setAlertMessage({
    type: "error",
    message,
  });
  return Promise.resolve();
};
```

화면에 에러 메시지를 toast 혹은 메시지로 노출 시키기 위해서 에러 내용을 store에 저장하는 코드입니다.



## Interceptors



### Request Interceptor

```typescript
const onRequest = (config: AxiosRequestConfig): AxiosRequestConfig => {
  const { method, url } = config;
  logOnDev(`🚀 [API] ${method?.toUpperCase()} ${url} | Request`);
  onLoading("request");
  if (method === "get") {
    config.params = {
      ...config.params,
      _t: Date.now(),
    };
    config.timeout = 15000;
  }
  return config;
};
```

[`AxiosRequestConfig`](https://axios-http.com/docs/req_config)는 axios request에서 config의 데이터를 담고 있는 패러미터입니다.

이를 이용하여 헤더의 내용을 수정할 수 있습니다.

혹은 인증과 관련하여 토큰을 넣는다거나 하는 행위를 이 `request interceptor`에서 합니다.



### Response Interceptor

#### Response

```typescript
const onResponse = (response: AxiosResponse): AxiosResponse => {
  const { method, url } = response.config;
  const { status } = response;
  logOnDev(`🚀 [API] ${method?.toUpperCase()} ${url} | Response ${status}`);
  onLoading("response");
  return response;
};
```

http response then으로 넘어가기 전에 call 되는 함수입니다.

#### Error

```typescript
const onErrorResponse = (error: AxiosError | Error): Promise<AxiosError> => {
  if (axios.isAxiosError(error)) {
    const { message } = error;
    const { method, url } = error.config as AxiosRequestConfig;
    const { statusText, status } = error.response as AxiosResponse;

    logOnDev(
      `🚨 [API] ${method?.toUpperCase()} ${url} | Error ${status} ${message}`
    );

    switch (status) {
      case 401: {
        onError("로그인이 필요합니다.");
        break;
      }
      case 403: {
        onError("권한이 없습니다.");
        break;
      }
      case 404: {
        onError("잘못된 요청입니다.");
        break;
      }
      case 500: {
        onError("서버에 문제가 발생했습니다.");
        break;
      }
      default: {
        onError("알 수 없는 오류가 발생했습니다.");
        break;
      }
    }
  } else {
    logOnDev(`🚨 [API] | Error ${error.message}`);
    onError(error.message);
  }

  onLoading("error");
  return Promise.reject(error);
};
```

http response catch으로 넘어가기 전에 call 되는 함수입니다.

에러 핸들링에서 메시지 내용이나, 코드, 처리 방식은 팀원마다 다를 수 있으니 그 때마다 맞춰서 작성하시면 되겠습니다.



## Setup Interceptors

```typescript
const setupInterceptors = (instance: AxiosInstance): AxiosInstance => {
  instance.interceptors.request.use(onRequest);
  instance.interceptors.response.use(onResponse, onErrorResponse);

  return instance;
};
```

`Axios Instance`에 interceptors의 내용을 등록하기 위한 코드입니다. 이 내용을 작성하지 않을 시 interceptors가 제대로 동작하지 않습니다.



