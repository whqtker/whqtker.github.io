---
title : "[Java] Jsoup 속도 개선 - ForkJoinPool / ForkJoin Framework"
date : 2025-03-26 11:00:00 +0900
categories : [Troubleshooting]
tags : [java, troubleshooting, spring boot, crawling, forkjoinpool, forkjoin framework, jsoup]
---

## 📌 문제의 코드

```java
    @Transactional
    public void saveTools() {
        List<SubCategory> subCategories = subCategoryRepository.findAll();
        
        for (SubCategory subCategory : subCategories) {
            String categoryUrl = subCategory.getSubCategoryUrl();
            int page = 1;

            while (true) {
                try {
                    String pageUrl = categoryUrl + "page/" + page + "/";
                    Document doc = Jsoup.connect(pageUrl).get();

                    Element latestPosts = doc.selectFirst("div[class='latest-posts']");
                    if (latestPosts == null) break;

                    Elements posts = latestPosts.select("div[class^=post-item]");
                    if (posts.isEmpty()) break;

                    for (Element post : posts) {
                        processPost(post);
                    }
                    page++;
                } catch (IOException e) {
                    log.error("페이지 크롤링 중 오류 발생: {} (카테고리: {})", e.getMessage(), subCategory.getSubCategoryName());
                    break;
                }
            }
        }
    }

    @Transactional
    protected void processPost(Element post) {
        Element titleElement = post.selectFirst("span.post-title a.dark-title");
        if (titleElement == null) return;

        String detailUrl = titleElement.attr("href");
        try {
            Document detailDoc = Jsoup.connect(detailUrl).get();
            String toolName = detailDoc.selectFirst("span[class*=post-title]").text();
            String description = detailDoc.selectFirst("span.desc-text").text();

            if (!toolsRepository.existsByToolName(toolName)) {
                Tool savedTool = saveTool(toolName, description, detailUrl);
                processCategories(detailDoc, savedTool);
            }
        } catch (IOException e) {
            log.error("도구 상세 페이지 크롤링 중 오류 발생: {}", e.getMessage());
        }
    }

    private Tool saveTool(String toolName, String description, String detailUrl) {
        Tool tool = Tool.builder()
                .toolName(toolName)
                .toolDescription(description)
                .toolLink(detailUrl)
                .toolCategories(new ArrayList<>())
                .build();

        Tool savedTool = toolsRepository.save(tool);
        log.info("저장된 AI 도구: {}", toolName);
        return savedTool;
    }

    private void processCategories(Document detailDoc, Tool savedTool) {
        Elements categoryElements = detailDoc.select("div.entry-categories a span[data-title]");
        for (Element categoryElement : categoryElements) {
            String categoryName = categoryElement.attr("data-title").trim();
            
            SubCategory subCat = subCategoryRepository.findBySubCategoryName(categoryName)
                    .orElse(null);

            if (subCat != null) {
                ToolCategory toolCategory = ToolCategory.builder()
                        .tool(savedTool)
                        .subCategory(subCat)
                        .build();

                savedTool.getToolCategories().add(toolCategory);
                if (subCat.getToolCategories() == null) {
                    subCat.setToolCategories(new ArrayList<>());
                }
                subCat.getToolCategories().add(toolCategory);

                toolCategoryRepository.save(toolCategory);
                
                log.info("도구-카테고리 연결: {} - {}", savedTool.getToolName(), categoryName);
            }
        }
    }
```

특정 웹사이트를 크롤링하는 코드이다. 간략하게 함수의 동작에 대해 알아보자.

- saveTools()

DB에 존재하는 SubCategory를 조회하여 각 SubCategory의 URL을 기반으로 페이지네이션하여 크롤링을 진행한다. 각 페이지에서 post를 수집한다.

- processPost()

수집한 post에 대하여 정보들을 수집한다.

- saveTool()

수집한 정보들을 토대로 Tool 엔티티를 생성하고 DB에 저장한다.

- processCategories()

Tool과 SubCategory 간 연관관계를 설정한다. ToolCategory 엔티티는 Tool과 SubCategory 간 다대다 관계를 풀기 위한 엔티티이다.

완전히 크롤링을 수행하기 걸리는 시간은 약 3시간이다. 주기적으로 크롤링을 하지 않을 예정이기 때문에 전체적인 서비스 측면에서는 굳이 성능 개선을 하지 않고 넘어가도 무방하다. 그러나 성능을 중요시 여기는 나로서는 그냥 넘어갈 수 없다. 병렬 처리를 사용하면 의미 있는 성능 개선이 이루어질 것이라고 생각했고, 이번 기회에 자바로 병렬 처리 코드를 작성하고 싶었다.

나는 `Fork/Join Framework`를 사용했다. 이번 기회를 통해 여러가지 병렬 처리 방법이 존재한다는 것을 알게 되었는데, Fork/Join Framework에 대한 내용을 정리하며 하나씩 살펴보려고 한다.

## 📌 개선된 코드

```java
    @Transactional
    public void saveTools() {
        List<SubCategory> subCategories = subCategoryRepository.findAll();
        
        // ForkJoinPool을 사용하여 병렬 처리
        ForkJoinPool customThreadPool = new ForkJoinPool(4);
        try {
            customThreadPool.submit(() ->
                subCategories.parallelStream().forEach(subCategory -> {
                    try {
                        processSubCategory(subCategory);
                    } catch (Exception e) {
                        log.error("카테고리 처리 중 오류 발생: {} (카테고리: {})", 
                            e.getMessage(), subCategory.getSubCategoryName());
                    }
                })
            ).get();
        } catch (Exception e) {
            log.error("병렬 처리 중 오류 발생: {}", e.getMessage());
        } finally {
            customThreadPool.shutdown();
        }
    }

    @Transactional
    protected void processSubCategory(SubCategory subCategory) {
        String categoryUrl = subCategory.getSubCategoryUrl();
        int page = 1;
        
        while (true) {
            try {
                String pageUrl = categoryUrl + "page/" + page + "/";
                Document doc = Jsoup.connect(pageUrl)
                        .timeout(10000)
                        .get();

                Element latestPosts = doc.selectFirst("div[class='latest-posts']");
                if (latestPosts == null) break;

                Elements posts = latestPosts.select("div[class^=post-item]");
                if (posts.isEmpty()) break;

                for (Element post : posts) {
                    try {
                        processPost(post);
                    } catch (Exception e) {
                        log.error("포스트 처리 중 오류 발생: {}", e.getMessage());
                    }
                }
                
                page++;
                Thread.sleep(1000);
                
            } catch (Exception e) {
                log.error("페이지 크롤링 중 오류 발생: {} (카테고리: {})", 
                    e.getMessage(), subCategory.getSubCategoryName());
                break;
            }
        }
    }

    @Transactional
    protected void processPost(Element post) {
        Element titleElement = post.selectFirst("span.post-title a.dark-title");
        if (titleElement == null) return;

        String detailUrl = titleElement.attr("href");
        try {
            Document detailDoc = Jsoup.connect(detailUrl)
                    .timeout(10000)
                    .get();
            String toolName = detailDoc.selectFirst("span[class*=post-title]").text();
            String description = detailDoc.selectFirst("span.desc-text").text();

            if (!toolsRepository.existsByToolName(toolName)) {
                Tool savedTool = saveTool(toolName, description, detailUrl);
                processCategories(detailDoc, savedTool);
            }
        } catch (IOException e) {
            log.error("도구 상세 페이지 크롤링 중 오류 발생: {}", e.getMessage());
        }
    }

    private Tool saveTool(String toolName, String description, String detailUrl) {
        Tool tool = Tool.builder()
                .toolName(toolName)
                .toolDescription(description)
                .toolLink(detailUrl)
                .toolCategories(new ArrayList<>())
                .build();

        Tool savedTool = toolsRepository.save(tool);
        log.info("저장된 AI 도구: {}", toolName);
        return savedTool;
    }

    @Transactional
    protected void processCategories(Document detailDoc, Tool savedTool) {
        Elements categoryElements = detailDoc.select("div.entry-categories a span[data-title]");
        for (Element categoryElement : categoryElements) {
            String categoryName = categoryElement.attr("data-title").trim();
            
            SubCategory subCat = subCategoryRepository.findBySubCategoryName(categoryName)
                    .orElse(null);

            if (subCat != null) {
                ToolCategory toolCategory = ToolCategory.builder()
                        .tool(savedTool)
                        .subCategory(subCat)
                        .build();

                toolCategory = toolCategoryRepository.save(toolCategory);
                
                savedTool.getToolCategories().add(toolCategory);
                
                log.info("도구-카테고리 연결: {} - {}", savedTool.getToolName(), categoryName);
            }
        }
    }
```

- saveTools()

```java
ForkJoinPool customThreadPool = new ForkJoinPool(4);
```

4개의 쓰레드로 구성된 pool을 생성한다.

```java
customThreadPool.submit(() ->
    subCategories.parallelStream().forEach(subCategory -> {
        processSubCategory(subCategory);
    })
).get();
```

편의상 로깅은 제거했다. `parallelStream()` 을 사용하여 병렬 스트림을 생성한다. 각 스트림에 대하여 processSubCategory()를 수행한다. `get()` 은 `submit()`으로 제출한 작업이 완료될 때까지 현재 쓰레드를 block하는 역할을 한다.

- processSubCategory()

```java
Thread.sleep(1000);
```

과도한 트래픽을 막기 위해 현재 쓰레드를 1초 동안 중지한다.

## 📌 성능 변화

기존 코드는 약 3시간 소요되었으나, 개선된 코드는 약 35분이 소요되었다. 약 **414%** 성능 개선 효과를 보았다.