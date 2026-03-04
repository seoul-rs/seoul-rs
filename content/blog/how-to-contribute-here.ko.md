+++
title = "seoul.rs에 글써보기"
date = "2026-03-04"
[taxonomies]
authors = ["Seungjin Kim"]
tags = ["howto"]
+++

seoul.rs는 [zola](https://getzola.org)라는 Rust로 작성된 SSG(Static Site Generator)를 기반으로 하며, [zola-bearblog](https://codeberg.org/alinnow/zola-bearblog) 템플릿을 사용하고 있습니다.

블로그(blog) 또는 소식(news)을 작성하려면 [github.com/seoul-rs/seoul-rs](https://github.com/seoul-rs/seoul-rs)를 포크한 후, content 폴더 아래의 blog 또는 news 디렉터리에 마크다운(.md) 파일을 생성하고, 아래 형식의 머리글을 추가한 뒤 작성하고자 하는 내용을 마크다운(Markdown) 형식에 맞게 작성하시면 됩니다.

```
+++
draft = true
title = "seoul.rs에 글써보기"
date = "2026-03-04"
[taxonomies]
authors = ["Seungjin Kim"]
tags = ["howto"]
+++

글쓰기 시작..

```

글 작성을 마친 후 github.com/seoul-rs/seoul-rs 저장소로 Pull Request를 보내주시면, 리뷰 후 병합(Merge)되며 GitHub Actions를 통해 자동으로 게시됩니다.

Pull Request를 생성할 때에는 본인의 암호화 키로 서명된 커밋만 허용하며, GitHub에 등록된 본인키로 검증(Verified)된 커밋만 병합하고 있습니다. GitHub 계정을 사용하지 않으시는 경우에는, 본인 GPG키로 서명된 패치 파일을 보내주시기 바랍니다.