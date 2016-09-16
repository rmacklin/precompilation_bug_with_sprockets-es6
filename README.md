Example app showing that using `register_mime_type` and `register_transformer`
instead of only `register_engine` causes files to get precompiled when they
shouldn't

---

I discovered that sprockets-es6 was precompiling `.es6` files that weren't
added to the precompile list. You can reproduce this by cloning this repo,
checking out commit
[b45e326](https://github.com/rmacklin/precompilation_bug_with_sprockets-es6/commit/b45e326d6315f57e566fd1cedb4393be92afed8a),
and running `rake assets:precompile` in the example app:

```sh
$ cd example_app
$ rake assets:precompile
$ ls public/assets
application-95ecf949959ffecaf38c5ab9b809b1ac7ec97f9c1b45f10b0371dd13aa88b986.js
application-95ecf949959ffecaf38c5ab9b809b1ac7ec97f9c1b45f10b0371dd13aa88b986.js.gz
application-e80e8f2318043e8af94dddc2adad5a4f09739a8ebb323b3ab31cd71d45fd9113.css
application-e80e8f2318043e8af94dddc2adad5a4f09739a8ebb323b3ab31cd71d45fd9113.css.gz
thing-1111dadedfdcb5c7e770d71736d2bdfc78ccfc450c6aa5e49657bc5b4955b575.es6
thing-1111dadedfdcb5c7e770d71736d2bdfc78ccfc450c6aa5e49657bc5b4955b575.es6.gz
```

Notice that [`thing.es6`](./example_app/app/assets/javascripts/thing.es6) is
compiled on its own, despite not having been added to the precompile list. Per
the [Rails guides](https://github.com/rails/rails/blame/47db952bf6938da78092096336d14c23b974a44c/guides/source/asset_pipeline.md#L731-L734):

> anything that compiles to JS/CSS is excluded [from precompilation]

so this shouldn't be happening. Indeed, it does not happen for coffeescript
files. But that statement should apply for _anything that compiles to JS_, not
just `.coffee` files.

The `CoffeeScriptProcessor` is registered via the deprecated `register_engine`
method: https://github.com/rails/sprockets/blob/v3.7.0/lib/sprockets.rb#L127

while the `ES6` processor in sprockets-es6 uses the new way of
`register_mime_type` and `register_transformer`:
https://github.com/TannerRogalsky/sprockets-es6/blob/fb498189d7d8b8a0bf19c215368265dadb443a99/lib/sprockets/es6.rb#L89-L90
though it still calls `register_engine` as well:
https://github.com/TannerRogalsky/sprockets-es6/blob/fb498189d7d8b8a0bf19c215368265dadb443a99/lib/sprockets/es6.rb#L97

I decided to fork sprockets-es6 and change this to _only_ use
`register_engine`: https://github.com/rmacklin/sprockets-es6/commit/e70ae24f150c7c7af7313d8d8ff83853ddb9dd8a

After switching to this fork
([e68421b](https://github.com/rmacklin/precompilation_bug_with_sprockets-es6/commit/e68421bdd9f45aca3ded224cda28c42787c98b0d)),
the `.es6` file was no longer precompiled:

```sh
$ cd example_app
$ rake assets:precompile
$ ls public/assets
application-95ecf949959ffecaf38c5ab9b809b1ac7ec97f9c1b45f10b0371dd13aa88b986.js
application-95ecf949959ffecaf38c5ab9b809b1ac7ec97f9c1b45f10b0371dd13aa88b986.js.gz
application-e80e8f2318043e8af94dddc2adad5a4f09739a8ebb323b3ab31cd71d45fd9113.css
application-e80e8f2318043e8af94dddc2adad5a4f09739a8ebb323b3ab31cd71d45fd9113.css.gz
```

Thus, it appears that using the new `register_mime_type` and
`register_transformer` methods leads to undesirable extra precompilation.
