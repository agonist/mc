---
title: "Android Jetpack Compose Linkify"
date: 2021-02-18T10:28:36+08:00
draft: true
---

# Linkify your text with Jetpack Compose

Before  Jetpack Compose, if you wanted to `Linkify` all links within a `TextView` it was quite straightforward. You only needed one line of code:  

```kotlin
Linkify.addLinks(myTextView, Linkify.EMAIL_ADDRESSES or Linkify.WEB_URLS)
```

But with Compose it's a different story. No more of this one magic line.

Here's what I ended up with to have my links working again:

```kotlin
@Composable
fun LinkifyText(text: String, modifier: Modifier = Modifier) {
    val uriHandler = AmbientUriHandler.current
    val layoutResult = remember {
        mutableStateOf<TextLayoutResult?>(null)
    }
    val linksList = extractUrls(text)
    val annotatedString = buildAnnotatedString {
        append(text)
        linksList.forEach {
            addStyle(
                    style = SpanStyle(
                            color = Color.Companion.Blue,
                            textDecoration = TextDecoration.Underline
                    ),
                    start = it.start,
                    end = it.end
            )
            addStringAnnotation(
                    tag = "URL",
                    annotation = it.url,
                    start = it.start,
                    end = it.end
            )
        }
    }
    Text(text = annotatedString, style = MaterialTheme.typography.body1, modifier = modifier.tapGestureFilter { offsetPosition ->
        layoutResult.value?.let {
            val position = it.getOffsetForPosition(offsetPosition)
            annotatedString.getStringAnnotations(position, position).firstOrNull()
                    ?.let { result ->
                        if (result.tag == "URL") {
                            uriHandler.openUri(result.item)
                        }
                    }
        }
    }, onTextLayout = { layoutResult.value = it })
}
```

It's mostly based on using [AnnotatedString](https://developer.android.com/reference/kotlin/androidx/compose/ui/text/AnnotatedString) which let you manipulate a Text style in a simple way.

The `extractUrls` function is a small utility function that find all the URL within the string.

```kotlin
private val urlPattern: Pattern = Pattern.compile(
        "(?:^|[\\W])((ht|f)tp(s?):\\/\\/|www\\.)"
                + "(([\\w\\-]+\\.){1,}?([\\w\\-.~]+\\/?)*"
                + "[\\p{Alnum}.,%_=?&#\\-+()\\[\\]\\*$~@!:/{};']*)",
        Pattern.CASE_INSENSITIVE or Pattern.MULTILINE or Pattern.DOTALL
)

fun extractUrls(text: String): List<LinkInfos> {
    val matcher = urlPattern.matcher(text)
    val links = arrayListOf<LinkInfos>()

    while (matcher.find()) {
        val matchStart = matcher.start(1)
        val matchEnd = matcher.end()

        var url = text.substring(matchStart, matchEnd)
        if (!url.startsWith("http://") && !url.startsWith("https://"))
            url = "https://$url"

        links.add(LinkInfos(url, matchStart, matchEnd))
    }
    return links
}

data class LinkInfos(
        val url: String,
        val start: Int,
        val end: Int
)
```

And that's it. You can now use this `LinkifyText` in your views.

```kotlin
val textWithLinks = "Eat Martabak and stay healthy -> https://indonesiaexpat.id/wp-content/uploads/2018/11/martabak-manis.jpg don't you think it looks good?"
LinkifyText(textWithLinks)
```
