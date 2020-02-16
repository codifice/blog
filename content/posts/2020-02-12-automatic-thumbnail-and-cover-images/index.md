---
title: "Automatic Thumbnail and Cover Images"
date: 2020-02-12T18:40:08Z
draft: false
authors: ["martincjarvis"]
comments: true
thumbnail: "/coding-gallery"
series: ["hugo"]
---
# Randomly Assign Cover Image from a Pool at Publish

In CMS terms the Cover image is a banner like image that appears towards the top of the article to provide some interest where the thumbnail image is used when linking to the article, such as the Recent Posts list on the Homepage.  

I prefer that these are related in someway to provide some visual context/confirmation for the user.  So that if they click on a link with a picture of a flower they should see the same flower immediately upon that page loading.

Unfortunately, most of the images that I'd be creating myself will be code related and pretty boring and I didn't want a manual chore to link specifc images.  So I decided to customise the site to enable a random high quality stock image to be assigned to each page when the site is published.  This image will then be the basis for an automatically generated Cover and Thumbnail image.

## How does it work? Layouts and Scratch

* Create themed Gallery Page Bundles to contain the relevant stock images
* Link posts to a particular gallery in the post "Front Matter" (metadata at the top of each content file), similar to `thumbnail: "/coding-gallery"` where "/coding-gallery" is the absolute path of the gallery page.
* Create layouts to select an image for the post from the related gallery, store it against the Post for reference and then generate the needed cropped and sized version of the image.

For example, the layout of the "Recent Posts" on the homepage is:

`/layouts/_default/box.html`
```html
{{- $original := . }}
{{- if (not ( $original.Scratch.Get "thumb-image")) and (not (.Params.cover)) and (.Params.thumbnail) -}}
  {{- if (not ( $original.Scratch.Get "image")) -}}
    {{$page := .Site.GetPage (printf "%s" .Params.thumbnail) }}

    {{- with $page.Resources -}}
      {{- with .Match "{*.jpg,*.png,*.jpeg}" -}}
        {{- with shuffle . -}}
          {{- range first 1 . -}}
            {{- $original.Scratch.Add "image" . -}}
          {{- end -}}
        {{- end -}}
      {{- end -}}
    {{- end -}}
  {{- end -}}

  {{- with $original.Scratch.Get "image" -}}
    {{- with .Fill "600x200" -}}
      {{- $original.Scratch.Add "thumb-image" ( printf "%s" .Permalink ) -}}
    {{- end -}}
  {{- end -}}
{{- else if .Params.cover }}
  {{ $original.Scratch.Add "thumb-image" .Params.cover }}
{{- end -}}

{{- $thumbimage := $original.Scratch.Get "thumb-image" -}}
<div class='item' style='background-image: url({{ $thumbimage }});'>
    <header class='item-header'>
    <h4 class='item-title'>
        <a href='{{ .Permalink | relURL }}'>
        {{- .Title -}}
        </a>
    </h4>
    </header>
</div>
```

To ensure that the same image is reused for each reference of the post we store the selected image reference against the in memory model of the post in the Scratch map.  We also cache resized file reference to prevent processing the same image multiple times during publish.
