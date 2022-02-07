# Part 3: Building a WebSite in Rust Using Rocket and Yew

Created: 02/04/2022

Simple Rust website with a yew frontend that interacts with a rocket backend Part3.

If you missed Part1 [Part 1](/posts/rust_rocket_yew_part1.md).
If you missed Part2 [Part 2](/posts/rust_rocket_yew_part2.md).

View the code for part 3 on [GitHub](https://github.com/nmharmon8/Rust_Rocket_Yew_Tutorial/tree/main/part3)

## Part 3

In Part3 we are going to send markup from the backend to the frontend. On the frontend we will render the markup into HTML and display it in the browser.

Pretty Cool 😃.

## Starting with The Data

In part 1 we create a project called common and then proceed to do nothing with it. Now it is time to build the common library. The common library will contain a data structure that both the frontend and backend can reference. The data structure will be serialized into Json by the backend and then sent to the frontend that will deserialize the json back into the data structure. This is all easy to do using the `serde` library we included in [Part 1](/posts/rust_rocket_yew_part1.md).

First lets add the serde depenedency to the common library.

___

```rust
[package]
name = "common"
version = "0.1.0"
edition = "2021"


[dependencies]
serde = {version = "1.0.133", feature = ["derive"]}
```

___

Now we can use serde to create a serializable data structure.
___

common/src/lib.rs

```rust
use serde::{Deserialize, Serialize};


#[derive(Debug, Clone, PartialEq, Default, Deserialize, Serialize)]
pub struct Markup {
    pub markup: String,
}
```

It is that easy. Define a data structure and add the Deserialize and Serialized macros. This is an over simple example but most rust primitives are supported by serde.

## Backend Data

We will start by creating a directory with markup data we want to serve to the frontend.

In the backend directory:

```bash
mkdir -p backend/data/markup
```

Lets add a new image into the image directory.

Save this image as `backend/images/tux.png`

<img src="/static/images/rust_website_tutorial/tux.png" width="300" />

Now lets create some markup to render on the homepage.

___
`backend/data/markup/homepage.md`

```rust
# This is Markup

## The markup is rendered as HTML and display on the homepage of the website.

![Image Displayed In Readme](/data/images/tux.png)
```

Learn more about CommonMark at: [commonmark.org](https://commonmark.org).

Notice that the markup references the image we just added. The existing route will already serve the tux image. Sweet!

## Serving Markup

Add a few more imports to the backend

___
backend/src/main.rs

```rust
//Import the data structure we just made
use common::Markup;
//Used to read files
use std::fs;
//Json for sending to the front end
use rocket::serde::json::Json;
```

___

Add one more route that will read the markup file into the common data structure and serve it as json.

___
backend/src/main.rs

```rust
#[get("/markup/<path..>")]
fn get_markup(path: PathBuf) -> Json<Markup> {
    let path = PathBuf::from("data/markup/").join(path);
    match fs::read_to_string(path) {
        Ok(markup) => Json(Markup {
            markup: String::from(markup),
        }),
        Err(e) => Json(Markup {
            markup: String::from(format!("Error: {}", e)),
        }),
    }
}
```

___

This route matches the `/markup/<path>` route. If the path matches a file in `data/markup` then it will return the file contents as a markup data structure serialized into json.

We are going to mount this route a little differently.

___
backend/src/main.rs

```rust
#[launch]
fn rocket() -> _ {
    rocket::build()
        .mount("/", routes![index, static_files, data])
        .mount("/api", routes![get_markup])
}
```

Now rather then matching /markup the route will match /api/markup. This make separation of functionality easy.

The backend is already to go 😛.

## Requesting and Rendering Markup

This gets a little more complicated but is great fun.

We will start by create new files in the UI:

```bash
mkdir -p ui/src/utils
mkdir -p ui/src/components
touch ui/src/utils/fetchstates.rs
touch ui/src/utils/mod.rs
touch ui/src/components/markup.rs
touch ui/src/components/mod.rs
```

___

We will start with the fetchstates which mange the state of the request to the backend for data. We have to keep track of what state a request for markup from the backend is in (not requested, requested, loaded, error). We will create some enums to represent the possible states and hold the data.

___
ui/src/utils/fetchstates.rs

```rust
use std::{
    error::Error,
    fmt::{self, Debug, Display, Formatter},
};

/// Something wrong has occurred while fetching an external resource.
#[derive(Debug, Clone, PartialEq)]
pub struct FetchError {
    pub err: String,
}

impl Display for FetchError {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        Debug::fmt(&self.err, f)
    }
}

impl Error for FetchError {}

/// The possible states a fetch request can be in.
#[derive(Debug, Clone, PartialEq)]
pub enum FetchState<T> {
    NotFetching,
    Fetching,
    Success(T),
    Failed(FetchError),
}

pub enum FetchStateMsg<T> {
    SetDataFetchState(FetchState<T>),
    GetData,
}
```

___

Make the fetchstates a public module by listing it in the mod.rs.

___
ui/src/utils/mod.rs

```rust
pub mod fetchstates;
```

Now we will create a new Component for loading and displaying Markup. The Markup Component will take a file name as input, request that markup from the backend and then render and display it as HTML. Rendering the markup into HTML will be done with pulldown-cmark which was added as a dependency in [Part 1](/posts/rust_rocket_yew_part1.md).

___
ui/src/components/markup.rs

```rust
use yew::prelude::*;

use pulldown_cmark::{html, Options, Parser};
use web_sys::Node;
use yew::virtual_dom::vnode::VNode;

use reqwasm::http::Request;

use crate::utils::fetchstates::{FetchError, FetchState, FetchStateMsg};

#[derive(Clone, Debug, Eq, PartialEq, Properties)]
pub struct Props {
    pub id: String,
}

pub struct Markup {
    markup: FetchState<common::Markup>,
}

impl Component for Markup {
    type Message = FetchStateMsg<common::Markup>;
    type Properties = Props;

    fn create(_ctx: &Context<Self>) -> Self {
        Self {
            markup: FetchState::NotFetching,
        }
    }

    fn changed(&mut self, ctx: &Context<Self>) -> bool {
        ctx.link().send_message(FetchStateMsg::GetData);
        true
    }

    fn update(&mut self, _ctx: &Context<Self>, _msg: Self::Message) -> bool {
        let uri: String = format!("/api/markup/{}", _ctx.props().id);

        match _msg {
            FetchStateMsg::SetDataFetchState(state) => {
                self.markup = state;
                true
            }
            FetchStateMsg::GetData => {
                _ctx.link().send_future(async move {
                    match Request::get(uri.as_str()).send().await {
                        Ok(makrup) => match makrup.json().await {
                            Ok(makrup) => {
                                FetchStateMsg::SetDataFetchState(FetchState::Success(makrup))
                            }
                            Err(err) => {
                                FetchStateMsg::SetDataFetchState(FetchState::Failed(FetchError {
                                    err: err.to_string(),
                                }))
                            }
                        },
                        Err(err) => {
                            FetchStateMsg::SetDataFetchState(FetchState::Failed(FetchError {
                                err: err.to_string(),
                            }))
                        }
                    }
                });
                _ctx.link()
                    .send_message(FetchStateMsg::SetDataFetchState(FetchState::Fetching));
                false
            }
        }
    }

    fn view(&self, _ctx: &Context<Self>) -> Html {
        if matches!(&self.markup, &FetchState::NotFetching) {
            _ctx.link().send_message(FetchStateMsg::GetData);
        }
        let div = web_sys::window()
            .unwrap()
            .document()
            .unwrap()
            .create_element("div")
            .unwrap();
        div.set_inner_html(&self.render_markdown());
        let node = Node::from(div);
        let vnode = VNode::VRef(node);
        vnode
    }
}

impl Markup {
    fn render_markdown(&self) -> String {
        let Self { markup } = self;
        let mut options = Options::empty();
        options.insert(Options::ENABLE_STRIKETHROUGH);

        match markup {
            FetchState::Success(markup) => {
                let markdown = markup.markup.clone();
                let parser = Parser::new_ext(&markdown[..], options);

                let mut html_output = String::new();
                html::push_html(&mut html_output, parser);
                html_output
            }
            _ => "Failed to Load ...".to_string(),
        }
    }
}
```

Clear as mud?

Lets break down the important parts.

```rust
fn update(&mut self, _ctx: &Context<Self>, _msg: Self::Message) -> bool {
    let uri: String = format!("/api/markup/{}", _ctx.props().id);

    match _msg {
        FetchStateMsg::SetDataFetchState(state) => {
            self.markup = state;
            true
        }
        FetchStateMsg::GetData => {
            _ctx.link().send_future(async move {
                match Request::get(uri.as_str()).send().await {
                    Ok(makrup) => match makrup.json().await {
                        Ok(makrup) => {
                            FetchStateMsg::SetDataFetchState(FetchState::Success(makrup))
                        }
                        Err(err) => {
                            FetchStateMsg::SetDataFetchState(FetchState::Failed(FetchError {
                                err: err.to_string(),
                            }))
                        }
                    },
                    Err(err) => {
                        FetchStateMsg::SetDataFetchState(FetchState::Failed(FetchError {
                            err: err.to_string(),
                        }))
                    }
                }
            });
            _ctx.link()
                .send_message(FetchStateMsg::SetDataFetchState(FetchState::Fetching));
            false
        }
    }
}
```

All the request logic for loading the markup from the backend is handled by the update method. Why is it so complicated? Browsers don't support blocking threads. So you can't just make a request to the backend and wait for a response.

Rather, we are starting a new thread `_ctx.link().send_future(async` that is going to get the data. At the bottom of the function. The original thread sends a message that we are currently fetching the data, and returns a false bool indicating that the Component should not rerender the view.

```rust
_ctx.link()
    .send_message(FetchStateMsg::SetDataFetchState(FetchState::Fetching));
false
```

Mean while in the new thread, we created, if the fetch is successful we send a message of success that contains the data.

```rust
FetchStateMsg::SetDataFetchState(FetchState::Success(makrup))
```

Where are the messages going? The messages trigger a new call to the update method. If the success message gets sent it triggers the first condition in the match statement.

```rust
 FetchStateMsg::SetDataFetchState(state) => {
        self.markup = state;
        true
    }
```

This statement sets the markup variable for the component with the data returned from the backend and returns the boolean true which triggers the component to rerender the view.

There is one more hacky thing happing in this component, and that is in the view function.

```rust
fn view(&self, _ctx: &Context<Self>) -> Html {
    if matches!(&self.markup, &FetchState::NotFetching) {
        _ctx.link().send_message(FetchStateMsg::GetData);
    }
    let div = web_sys::window()
        .unwrap()
        .document()
        .unwrap()
        .create_element("div")
        .unwrap();
    div.set_inner_html(&self.render_markdown());
    let node = Node::from(div);
    let vnode = VNode::VRef(node);
    vnode
}
```

The `render_markdown()` function returns the HTML version of the markup as a String. Right now there is not a nice way to convert an HTML string into the type Html. Rather we have to do that garbage 🤷.

Don't forget to to make the new Markup Component public.

___
ui/src/components/mod.rs

```rust
pub mod markup;
```

Now we just need to add the markup component in a few places. First let add the imports to the main.rs

ui/src/main.rs

```rust
mod components;
mod utils;
```

Now lets edit the homepage to render the markup rather then static text.

ui/src/pages/home.rs

```rust
use crate::components::markup::Markup;


fn view(&self, _ctx: &Context<Self>) -> Html {
    html! {
        <div class="justify-content-center m-5">

            <div class="d-flex justify-content-center m-5">
                <h1 class="text-truncate">{"Building a Website in Rust"}</h1>
            </div>

            <div class="container-sm justify-content-center m-5">
                <Markup id={"homepage.md"}/>
            </div>
        </div>
    }
}
```

<img src="/static/images/rust_website_tutorial/final.png" width="600" />

All done. You now have the basic building blocks to do something cool. Have fun building.

If you had issues don't forget to create issues on [GitHub](https://github.com/nmharmon8/Rust_Rocket_Yew_Tutorial)
