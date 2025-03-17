# tmp

```java
EventSourceListener listener = new EventSourceListener() {
                private StringBuilder buffer = new StringBuilder();

                @Override
                public void onOpen(@NotNull EventSource eventSource, @NotNull Response response) {
                    log.debug("conversation open. Url = {}", eventSource.request().url());
                }

                @Override
                public void onClosed(@NotNull EventSource eventSource) {
                    log.info("code completion close cost: {}ms", System.currentTimeMillis() - l);
                    sink.complete();
                    isComplete.set(true);
                }

                @Override
                public void onEvent(@NotNull EventSource eventSource, @Nullable String id, @Nullable String type, @NotNull String data) {
                    if (StringUtils.isBlank(data)) {
                        return;
                    }
                    log.debug("conversation data. Url = {}", eventSource.request().url());
                    ModelServerCodeCompletionResponseForDbs response = GsonUtil.fromJson(data, ModelServerCodeCompletionResponseForDbs.class);
                    last.append(response.getCode() + "\n");
                    sink.next(response);
                }

                @Override
                public void onFailure(@NotNull EventSource eventSource, @Nullable Throwable t, @Nullable Response response) {
                    sink.error(t == null ? new RuntimeException("conversation error") : t);
                    if (isComplete.compareAndSet(false, true)) {
                        log.warn("conversation error. Url = {}", eventSource.request().url(), t);
                    }
                }
            };
            EventSource.Factory factory = EventSources.createFactory(okHttpChatClient);
            EventSource source = factory.newEventSource(httpRequest, listener);
```
