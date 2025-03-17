# tmp

```java
package kuaishou.kwaipilot.onprem.service.completion;

import lombok.extern.slf4j.Slf4j;
import okhttp3.*;
import okhttp3.sse.EventSource;
import okhttp3.sse.EventSourceListener;
import okhttp3.sse.EventSources;
import org.apache.commons.lang3.StringUtils;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import javax.net.ssl.*;
import java.io.IOException;
import java.security.KeyManagementException;
import java.security.NoSuchAlgorithmException;
import java.security.SecureRandom;
import java.security.cert.X509Certificate;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Create on 2025/3/17
 */
@Slf4j
public class CodeCompletinoDemoCall {

    private static final OkHttpClient okHttpChatClient;

    private static final String completionUrl = "/mapi/code/completion";

    private static final String host = "http://localhost:8080";

    /**
     * Get initialized SSLContext instance which ignored SSL certification
     *
     * @return
     * @throws NoSuchAlgorithmException
     * @throws KeyManagementException
     */
    public static SSLContext getIgnoreInitedSslContext() throws NoSuchAlgorithmException, KeyManagementException {
        var sslContext = SSLContext.getInstance("SSL");
        sslContext.init(null, new TrustManager[]{IGNORE_SSL_TRUST_MANAGER_X509}, new SecureRandom());
        return sslContext;
    }

    /**
     * X509TrustManager instance which ignored SSL certification
     */
    public static final X509TrustManager IGNORE_SSL_TRUST_MANAGER_X509 = new X509TrustManager() {
        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType) {
        }

        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType) {
        }

        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return new X509Certificate[]{};
        }
    };

    public static HostnameVerifier getIgnoreSslHostnameVerifier() {
        return new HostnameVerifier() {
            @Override
            public boolean verify(String arg0, SSLSession arg1) {
                return true;
            }
        };
    }

    static {
        try {
            Dispatcher dispatcher = new Dispatcher();
            dispatcher.setMaxRequests(400);
            dispatcher.setMaxRequestsPerHost(200);

            okHttpChatClient = new OkHttpClient
                    .Builder()
                    .sslSocketFactory(getIgnoreInitedSslContext().getSocketFactory(), IGNORE_SSL_TRUST_MANAGER_X509)
                    .hostnameVerifier(getIgnoreSslHostnameVerifier())
                    .dispatcher(dispatcher)
                    .connectTimeout(2000, TimeUnit.MILLISECONDS)
                    .readTimeout(5000, TimeUnit.MILLISECONDS)
                    .build();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public SseEmitter codeCompletion(String requestBody, String encryption) {
        SseEmitter emitter = new SseEmitter();
        AtomicBoolean isComplete = new AtomicBoolean(false);
        Request httpRequest = new Request
                .Builder()
                .header("Encryption-Key", encryption)
                .url(host + completionUrl)
                .post(RequestBody.create(MediaType.parse("application/json"), requestBody)).build();
        EventSourceListener listener = new EventSourceListener() {

            private StringBuilder buffer = new StringBuilder();

            @Override
            public void onOpen(@NotNull EventSource eventSource, @NotNull Response response) {
                log.debug("conversation open. Url = {}", eventSource.request().url());
            }

            @Override
            public void onClosed(@NotNull EventSource eventSource) {
                log.info("code completion close.");
            }

            @Override
            public void onEvent(@NotNull EventSource eventSource, @Nullable String id, @Nullable String type, @NotNull String data) {
                if (StringUtils.isBlank(data)) {
                    return;
                }
                log.debug("conversation data: {}", data);
                try {
                    emitter.send(data);
                } catch (IOException e) {
                    isComplete.set(true);
                    log.warn("conversation data cancel");
                }
            }

            @Override
            public void onFailure(@NotNull EventSource eventSource, @Nullable Throwable t, @Nullable Response response) {
                if (isComplete.compareAndSet(false, true)) {
                    log.warn("conversation error. Url = {}", eventSource.request().url(), t);
                }
            }
        };
        EventSource.Factory factory = EventSources.createFactory(okHttpChatClient);
        EventSource source = factory.newEventSource(httpRequest, listener);
        return emitter;
    }
}
```
