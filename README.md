package com.example;

import org.jsoup.Jsoup;
import org.jsoup.nodes.Document;
import org.jsoup.nodes.Element;
import org.jsoup.select.Elements;

import java.io.IOException;
import java.net.URI;
import java.util.HashSet;
import java.util.LinkedList;
import java.util.Queue;
import java.util.Set;

public class SimpleCrawler {

    // 最多抓取页面数
    private static final int MAX_PAGES = 10;

    // 请求超时时间
    private static final int TIMEOUT_MS = 10000;

    // 每次请求间隔，避免爬太快
    private static final long DELAY_MS = 500;

    private static final String USER_AGENT =
            "Mozilla/5.0 (Windows NT 10.0; Win64; x64) " +
            "AppleWebKit/537.36 (KHTML, like Gecko) " +
            "Chrome/120.0 Safari/537.36";

    public static void main(String[] args) {
        // 改成你要爬取的起始网址
        String startUrl = "https://example.com";
        crawl(startUrl);
    }

    public static void crawl(String startUrl) {
        URI startUri;
        try {
            startUri = URI.create(startUrl);
        } catch (IllegalArgumentException e) {
            System.err.println("起始 URL 不合法: " + startUrl);
            return;
        }

        String host = startUri.getHost();

        Queue<String> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();

        queue.offer(startUrl);

        int count = 0;

        while (!queue.isEmpty() && count < MAX_PAGES) {
            String url = queue.poll();

            if (url == null || visited.contains(url)) {
                continue;
            }

            visited.add(url);
            count++;

            try {
                Document doc = Jsoup.connect(url)
                        .userAgent(USER_AGENT)
                        .timeout(TIMEOUT_MS)
                        .get();

                System.out.println("\n[" + count + "] " + url);
                System.out.println("标题: " + doc.title());

                // 提取页面中的 h1
                Elements h1s = doc.select("h1");
                for (Element h1 : h1s) {
                    System.out.println("H1: " + h1.text());
                }

                // 提取所有链接
                Elements links = doc.select("a[href]");
                for (Element link : links) {
                    String absUrl = link.absUrl("href");

                    if (absUrl.isEmpty()) {
                        continue;
                    }

                    // 去掉锚点
                    absUrl = absUrl.split("#")[0];

                    if (absUrl.isEmpty()) {
                        continue;
                    }

                    try {
                        URI u = URI.create(absUrl);
                        String scheme = u.getScheme();

                        // 只处理 http / https
                        if (scheme == null || !(scheme.equals("http") || scheme.equals("https"))) {
                            continue;
                        }

                        // 只爬同一个域名，避免爬出站
                        if (host != null && !host.equals(u.getHost())) {
                            continue;
                        }

                        if (!visited.contains(absUrl)) {
                            queue.offer(absUrl);
                        }

                    } catch (IllegalArgumentException ignored) {
                        // 忽略非法链接
                    }
                }

                // 控制爬取速度
                Thread.sleep(DELAY_MS);

            } catch (IOException e) {
                System.err.println("抓取失败: " + url + "，原因: " + e.getMessage());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                System.err.println("爬虫被中断");
                break;
            }
        }

        System.out.println("\n完成，共抓取 " + count + " 个页面。");
    }
}
