# Environment Setup

Before we begin we need to set up our environment.

## Virtualenv

You should always use virtualenv when using Python. It allows you created a sanboxed Python environment and avoid conflicts that would occur if you loaded your Python libraries into you global environment. If I'm building an app, I'll create a virtual environemnt for each Python app. For research, however, I like to use `virtualenvwrapper` which let's you create a "global" virtual environment. Don't worry if that's confusing, I'll walk you through it.

### Install Virtualenv

Install virtualenv via pip:

```
    $ pip install virtualenv
```

Test your installation

```
    $ virtualenv --version
```

You should see a version number. I'm working with `15.0.3`

### Using Virtualenv

Once you have virtualenv installed, you first create a new directory and then instantiate a virtual environment in that directory. 

```
    $ mkdir example_app_folder
    $ cd example_app_folder
    $ virtualenv example_app
```




### Install Dependencies

After you have virtualenv up and running. 

I'm going to be using [Keras](https://keras.io/) with [Tensor Flow](https://www.tensorflow.org/) as the backend. Keras doesn't support Tensor Flow 1.0 unless you install Keras 2.0 which is a real pain to install so we'll use Keras with Tensor Flow 0.12.  You'll need Python and Postgresql installed on your machine. In the github repo, you'll find a `requirements.txt` with the dependencies you need. If you don't use it already, you should also load virtualenv to sandbox your Python environment.

