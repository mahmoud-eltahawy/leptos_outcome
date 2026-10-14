# content of file at path : ./Cargo.toml
```toml
    [package]
    name = "leptos-outcome"
    version = "0.1.0"
    edition = "2024"
    description = "Outcome-oriented ResourceView and ErrorView components for Leptos"
    keywords = ["leptos", "error", "resource", "async", "ui"]
    categories = ["web-programming", "wasm"]
    
    [dependencies]
    leptos = "0.8"
    serde = { version = "1", features = ["derive"] }
```
# content of file at path : ./review.md
```md

```
# content of file at path : ./README.md
```md
    # leptos-outcome
    
    Outcome-oriented `ResourceView` and `ErrorView` components for [Leptos](https://leptos.dev).
    
    Most Leptos apps end up writing the same three things over and over:
    
    1. A `match` around every `Resource<Result<T, E>>` for loading / error / success.
    2. The same error card in every action's failure branch.
    3. Some ad-hoc trait or convention to tell a "retryable" error from a fatal one.
    
    `leptos-outcome` packages all three into two components and one trait:
    
    - **`ResourceView`** — renders `Resource<Result<T, E>>` as loading, success, or error.
    - **`ErrorView`** — renders a `UserFacingError` in `Inline` or `Banner` style.
    - **`UserFacingError`** — the trait that drives both, decoupling the error type from the UI.
    
    ## Example
    
    ```rust,ignore
    use leptos::prelude::*;
    use leptos_outcome::{Disposition, ErrorView, ResourceView, UserFacingError};
    
    #[derive(Clone)]
    struct AppError { message: String, retryable: bool }
    
    impl UserFacingError for AppError {
        fn message(&self) -> String { self.message.clone() }
        fn disposition(&self) -> Disposition {
            if self.retryable { Disposition::Transient } else { Disposition::Fatal }
        }
    }
    
    #[component]
    fn Page() -> impl IntoView {
        let items = Resource::new(|| (), |_| fetch_items());
        view! { <ResourceView resource=items view_fn=ItemsList adapter=|v| v.into()/> }
    }
```
# content of file at path : ./src/error.rs
```rs
    use leptos::{either::Either, prelude::*};
    use serde::{Deserialize, Serialize};
    
    use crate::icons::ErrorIcon;
    
    // ─── Trait ────────────────────────────────────────────────────────────────────
    
    /// The error-side contract that drives [`ErrorView`] and [`ResourceView`].
    ///
    /// Implement this on whatever error type your server functions return. The
    /// trait has two methods on purpose:
    ///
    /// - [`message`](Self::message) is the text the user reads. It should be safe
    ///   to display verbatim — no diagnostics, no stack traces.
    /// - [`disposition`](Self::disposition) tells the UI whether retrying the same
    ///   operation could succeed.
    ///
    /// The type must be `Clone` (component props are signals, which clone
    /// frequently), `Send + Sync` (Leptos resources cross threads under SSR), and
    /// `'static`.
    pub trait UserFacingError: Clone + Send + Sync + 'static {
        /// Text shown to the user. Should be safe to display verbatim.
        fn message(&self) -> String;
    
        /// Whether retrying the same operation could succeed.
        fn is_fatal(&self) -> bool;
    }
    
    // ─── SimpleError ──────────────────────────────────────────────────────────────
    
    /// A minimal [`UserFacingError`] for hosts that don't have a dedicated error
    /// type yet.
    ///
    /// Use this when you're prototyping, or in tests, or anywhere a one-off error
    /// needs to flow through the same pipeline as your real one.
    #[derive(Clone, Debug, Serialize, Deserialize)]
    pub struct SimpleError {
        /// The user-facing message.
        pub message: String,
        /// Whether retrying the same operation could succeed.
        pub fatal: bool,
    }
    
    impl SimpleError {
        /// A retryable error with the given message.
        pub fn new_transient(message: impl Into<String>) -> Self {
            Self {
                message: message.into(),
                fatal: false,
            }
        }
    
        /// An error that will produce the same result on retry.
        pub fn new_fatal(message: impl Into<String>) -> Self {
            Self {
                message: message.into(),
                fatal: true,
            }
        }
    }
    
    impl UserFacingError for SimpleError {
        fn message(&self) -> String {
            self.message.clone()
        }
    
        fn is_fatal(&self) -> bool {
            self.fatal
        }
    }
    
    impl std::fmt::Display for SimpleError {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            f.write_str(&self.message)
        }
    }
    
    impl std::error::Error for SimpleError {}
    
    // ─── Component ────────────────────────────────────────────────────────────────
    
    /// Render a [`UserFacingError`] consistently.
    ///
    /// Renders nothing when `error` is `None`. When `Some(e)`, reads `e.message()`
    /// and decides whether to show a retry control based on `e.disposition()`:
    ///
    /// * `Transient` — retry offered, if `on_retry` is provided.
    /// * `Fatal` — no retry. The click would produce the same result.
    ///
    /// # Example
    ///
    /// ```rust,ignore
    /// <ErrorView
    ///     error=action_error(submit)
    ///     on_retry=Callback::new(move |_| submit.dispatch(()))
    ///     banner=true
    ///     heading="Upload failed"
    /// />
    /// ```
    #[component]
    pub fn ErrorView<E>(
        /// The error to render, or `None` to render nothing.
        error: Signal<Option<E>>,
        /// Called when the user clicks "Retry". Only shown for `Transient` errors.
        #[prop(optional, into)]
        on_retry: Option<Callback<()>>,
        /// Visual treatment. Defaults to [`ErrorStyle::Inline`].
        #[prop(default = false)]
        banner: bool,
        /// Override the default `Banner` heading. Ignored for `Inline`.
        #[prop(optional, into)]
        heading: Option<String>,
    ) -> impl IntoView
    where
        E: UserFacingError + 'static,
    {
        move || {
            let e = error.get()?;
            let retry = if !e.is_fatal() { on_retry } else { None };
    
            Some(match banner {
                false => Either::Right(inline_view(e.message(), retry)),
                true => Either::Left(banner_view(e.message(), retry, heading.clone())),
            })
        }
    }
    
    // ─── Renderers ────────────────────────────────────────────────────────────────
    
    fn inline_view(msg: String, retry: Option<Callback<()>>) -> impl IntoView {
        view! {
            <div class="text-red-300 text-sm flex items-center gap-2">
                <span>{msg}</span>
                {retry.map(|cb| view! {
                    <button
                        type="button"
                        on:click=move |_| cb.run(())
                        class="underline hover:text-red-200 transition-colors text-xs shrink-0"
                    >
                        "retry"
                    </button>
                })}
            </div>
        }
    }
    
    fn banner_view(msg: String, retry: Option<Callback<()>>, heading: Option<String>) -> impl IntoView {
        let heading = heading.unwrap_or("Something went wrong".to_string());
    
        view! {
            <div class="bg-red-500/15 border border-red-500/30 rounded-xl p-4 space-y-2">
                <div class="flex items-start gap-3">
                    <span class="text-red-400 shrink-0 mt-0.5"><ErrorIcon/></span>
                    <div class="flex-1 min-w-0">
                        <div class="text-red-300 text-sm font-bold">{heading}</div>
                        <div class="text-red-300/80 text-xs break-all mt-1">{msg}</div>
                    </div>
                </div>
                {retry.map(|cb| view! {
                    <div class="flex justify-end">
                        <button
                            type="button"
                            on:click=move |_| cb.run(())
                            class="text-xs px-3 py-1 rounded-lg bg-red-500/20 \
                                       hover:bg-red-500/30 text-red-200 transition"
                        >
                            "Retry"
                        </button>
                    </div>
                })}
            </div>
        }
    }
    
    // ─── Tests ────────────────────────────────────────────────────────────────────
    
    #[cfg(test)]
    mod tests {
        use super::*;
    
        #[test]
        fn simple_error_transient() {
            let e = SimpleError::new_transient("retry me");
            assert_eq!(e.message(), "retry me");
            assert!(!e.is_fatal());
        }
    
        #[test]
        fn simple_error_fatal() {
            let e = SimpleError::new_fatal("nope");
            assert_eq!(e.message(), "nope");
            assert!(e.is_fatal());
        }
    
        #[test]
        fn simple_error_displays_as_message() {
            let e = SimpleError::new_fatal("bad input");
            assert_eq!(e.to_string(), "bad input");
        }
    }
```
# content of file at path : ./src/extract.rs
```rs
    use leptos::prelude::*;
    use serde::Serialize;
    use serde::de::DeserializeOwned;
    
    /// Pull the error out of an [`Action<_, Result<V, E>>`](leptos::prelude::Action).
    ///
    /// Returns a signal that is `Some(e)` whenever the action's last result was
    /// `Err(e)`, and `None` otherwise. Bind it directly to
    /// [`ErrorView`](crate::ErrorView).
    ///
    /// # Example
    ///
    /// ```rust,ignore
    /// let submit = Action::new(|t: &String| create(t.clone()));
    /// view! { <ErrorView error=action_error(submit)/> }
    /// ```
    pub fn action_error<A, V, E>(action: Action<A, Result<V, E>>) -> Signal<Option<E>>
    where
        A: Send + Sync + 'static,
        V: Send + Sync + Clone + 'static,
        E: Send + Sync + Clone + 'static,
    {
        Signal::derive(move || action.value().get().and_then(|r| r.err()))
    }
    
    /// Pull the error out of a [`Resource<Result<T, E>>`](leptos::prelude::Resource).
    ///
    /// Returns `Some(e)` when the resource has resolved to `Err(e)`, and `None`
    /// while it is still loading or has resolved successfully.
    pub fn resource_error<T, E>(resource: Resource<Result<T, E>>) -> Signal<Option<E>>
    where
        T: Serialize + DeserializeOwned + Send + Sync + Clone + 'static,
        E: Send + Sync + Clone + 'static,
    {
        Signal::derive(move || resource.get().and_then(|r| r.err()))
    }
    
    /// Pull the error out of a hand-rolled `Signal<Option<Result<T, E>>>`.
    ///
    /// Useful when the error source isn't a Leptos `Action` or `Resource` —
    /// for instance when it's threaded through your own state machine.
    pub fn result_signal_error<T, E>(result: Signal<Option<Result<T, E>>>) -> Signal<Option<E>>
    where
        T: Send + Sync + Clone + 'static,
        E: Send + Sync + Clone + 'static,
    {
        Signal::derive(move || result.get().and_then(|r| r.err()))
    }
```
# content of file at path : ./src/lib.rs
```rs
    //! Outcome-oriented views for Leptos.
    //!
    //! Two components and one trait:
    //!
    //! - [`ResourceView`] — render a [`Resource<Result<T, E>>`](leptos::prelude::Resource)
    //!   as loading, success, or error.
    //! - [`ErrorView`] — render an error the same way everywhere, in `Inline` or
    //!   `Banner` style.
    //! - [`UserFacingError`] — the trait that drives both.
    //!
    //! # Getting started
    //!
    //! ```rust
    //! use leptos_outcome::{ErrorView, UserFacingError};
    //!
    //! #[derive(Clone)]
    //! pub struct AppError {
    //!     pub message: String,
    //!     pub is_fatal: bool,
    //! }
    //!
    //! impl UserFacingError for AppError {
    //!     fn message(&self) -> String {
    //!         self.message.clone()
    //!     }
    //!
    //!     fn is_fatal(&self) -> bool {
    //!         self.is_fatal
    //!     }
    //! }
    //! ```
    //!
    //! # Design
    //!
    //! The crate has no opinion about what an error *is*. It only asks that the
    //! error type be cheap to clone (component props are signals), `Send + Sync`
    //! (Leptos resources cross threads under SSR), and able to answer two
    //! questions:
    //!
    //! 1. What should the user read?
    //! 2. Is retrying worth offering?
    #![forbid(unsafe_code)]
    
    pub mod error;
    pub mod extract;
    pub mod icons;
    pub mod resource_view;
    
    // ─── Root re-exports ──────────────────────────────────────────────────────────
    
    pub use error::{ErrorView, SimpleError, UserFacingError};
    pub use extract::{action_error, resource_error, result_signal_error};
    pub use resource_view::ResourceView;
```
# content of file at path : ./src/resource_view.rs
```rs
    use std::sync::Arc;
    
    use leptos::prelude::*;
    use serde::Serialize;
    use serde::de::DeserializeOwned;
    
    use crate::error::{ErrorView, UserFacingError};
    use crate::icons::LoadingIcon;
    
    /// Render a [`Resource<Result<T, E>>`](leptos::prelude::Resource) as loading,
    /// success, or error.
    ///
    /// On success it calls `adapter(value)` and feeds the result into `view_fn` —
    /// this split exists so the underlying view can be a normal `#[component]` with
    /// explicit props, while `ResourceView` supplies those props from whatever the
    /// resource resolved to.
    ///
    /// On error it renders [`ErrorView`] with a retry that refetches the resource.
    ///
    /// While loading it renders `fallback`, or a spinner if none is given.
    ///
    /// # Example
    ///
    /// ```rust,ignore
    /// #[component]
    /// fn ItemsList(items: Vec<Item>) -> impl IntoView { /* … */ }
    ///
    /// let items: Resource<Result<Vec<Item>, AppError>> =
    ///     Resource::new(id, fetch_items);
    ///
    /// view! {
    ///     <ResourceView
    ///         resource=items
    ///         view_fn=ItemsList
    ///         adapter=|v: Vec<Item>| v
    ///     />
    /// }
    /// ```
    #[component]
    pub fn ResourceView<ResourceViewFn, ResourceValue, ViewValue, Props, Adapter, E>(
        /// The resource to render.
        resource: Resource<Result<ResourceValue, E>>,
        /// The component that renders the success value.
        view_fn: ResourceViewFn,
        /// Maps the resource's success value into the props for `view_fn`.
        adapter: Adapter,
        /// Optional loading fallback. Defaults to a spinner.
        #[prop(optional, into)]
        fallback: Option<ViewFn>,
    ) -> impl IntoView
    where
        ResourceValue: Serialize + DeserializeOwned + Clone + Send + Sync + 'static,
        ViewValue: IntoView + Send + 'static,
        ResourceViewFn: Fn(Props) -> ViewValue + Send + Sync + 'static,
        Props: Send + 'static,
        Adapter: Fn(ResourceValue) -> Props + Send + Sync + 'static,
        E: UserFacingError + Serialize + DeserializeOwned + Clone + Send + Sync + 'static,
    {
        let view_fn = Arc::new(view_fn);
        let adapter = Arc::new(adapter);
    
        let core = move || {
            let res = resource.get()?;
            Some(match res {
                Ok(val) => view_fn(adapter(val)).into_any(),
                Err(e) => {
                    let err_signal = Signal::derive(move || Some(e.clone()));
                    let retry = Callback::new(move |_| resource.refetch());
                    view! {
                        <div class="py-8 text-center">
                            <ErrorView
                                error=err_signal
                                on_retry=retry
                                banner=true
                            />
                        </div>
                    }
                    .into_any()
                }
            })
        };
    
        let fallback_closure = {
            let fallback = fallback.clone();
            move || match fallback.clone() {
                Some(f) => f.run(),
                None => view! { <LoadingIcon/> }.into_any(),
            }
        };
    
        view! {
            <Suspense fallback=fallback_closure>
                {core}
            </Suspense>
        }
    }
```
# content of file at path : ./src/icons.rs
```rs
    //! The two icons the crate renders itself.
    //!
    //! They are public so hosts that want visual consistency can reuse them, but
    //! the crate treats them as an implementation detail: a future version may
    //! swap them out or add a customization hook.
    
    use leptos::prelude::*;
    
    #[component]
    fn Icon(
        children: Children,
        #[prop(into)] class: String,
        #[prop(into)] label: String,
        #[prop(default = false)] filled: bool,
        #[prop(default = "1.5")] stroke_width: &'static str,
    ) -> impl IntoView {
        view! {
            <svg
                xmlns="http://www.w3.org/2000/svg"
                class=class
                viewBox="0 0 24 24"
                fill=if filled { "currentColor" } else { "none" }
                stroke=if !filled { "currentColor" } else { "none" }
                stroke-width=stroke_width
                stroke-linecap="round"
                stroke-linejoin="round"
                role="img"
                aria-label=label.clone()
            >
                {children()}
            </svg>
        }
    }
    
    /// A red circle-with-exclamation icon. Used by [`ErrorView`](crate::ErrorView)'s
    /// banner style.
    #[component]
    pub fn ErrorIcon() -> impl IntoView {
        view! {
            <Icon class="h-6 w-6 text-red-400" label="Something Went Wrong">
                <circle cx="12" cy="12" r="10" />
                <line x1="12" y1="8" x2="12" y2="12" />
                <circle cx="12" cy="16" r="0.5" fill="currentColor" />
            </Icon>
        }
    }
    
    /// A spinning cyan arc. Used by [`ResourceView`](crate::ResourceView) as the
    /// default loading fallback.
    #[component]
    pub fn LoadingIcon() -> impl IntoView {
        view! {
            <Icon class="h-8 w-8 animate-spin text-cyan-400" label="Loading">
                <circle cx="12" cy="12" r="10" stroke-dasharray="70 200" stroke-dashoffset="0" />
                <circle cx="12" cy="12" r="2" fill="currentColor" />
            </Icon>
        }
    }
```

