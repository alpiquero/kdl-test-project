# KDL Project Template

## Project structure

The project repository has the following directory structure:

```
├── .vscode
│   └── launch.json <- Configuration for test executions in Vscode
│
├── goals         <- Acceptance criteria (typically as automated tests describing desired behaviour)
│
├── lab
│   │
│   ├── analysis  <- Analyses of data, models etc. (typically notebooks)
│   │
│   ├── docs      <- High-level reports, executive summaries at each milestone (typically .md)
│   │
│   └── processes           <- Source code for reproducible workflow steps.
│       ├── prepare_data
│       │   ├── main.py
│       │   ├── cancer_data.py
│       │   └── cancer_data_test.py
|       ├── train_dnn_pytorch
│       │   ├── main.py
│       │   ├── densenet.py
│       │   └── densenet_test.py
│       └── train_standard_classifiers
│       │   ├── main.py
│       │   ├── classifiers.py
│       │   └── classifiers_test.py
│       │
│       ├── config.ini         <- Config for Drone runs
│       ├── config_test.ini   <- Config for local test runs
│       └── conftest.py        <- Pytest fixtures
|
├── lib           <- Importable functions used by analysis notebooks and processes scripts
|
├── runtimes      <- Code for generating deployment runtimes (.krt)
│
├── .drone.yml    <- Instructions for Drone runners
├── .env          <- Local environment variables for VScode IDE
├── .gitignore    
├── README.md     <- Main README
└── pytest.ini    <- Pytest configuration
```

The `processes` subdirectory contains as its subdirectories the various separate processes (`prepare_data`, etc.),
which can be tought of as nodes of an analysis graph.
Each of these processes contains:
- `main.py`, a clearly identifiable main script for running on CI/CD (Drone)
- `{process}.py`, containing importable functions and classes specific to that process,
- `{process}_test.py`, containing automated unit or integration tests for this process, and

The process names from the template are not likely to generalize to other projects, so here is another example for clarity:

```
└── processes
    ├── prepare_data
    │   ├── main.py
    │   ├── (image_data).py         <- importable functions
    │   └── (image_data)_test.py    <- for automated testing
    ├── train_model
    │   ├── main.py
    │   ├── (convnet).py
    │   └── (convnet)_test.py
    └── ...
```

In the examples shown, all processes files are Python `.py` files.
However, the idea of modularizing the analysis into separate processes facilitates changing any of those processes to a different language as may be required, for example R or Julia.

## Example project pipeline

KDL contains various components that need to be correctly orchestrated and connected.
To illustrate their intended usage, we provide an example machine learning pipeline already implemented in KDL.

The example pipeline is a simple classification problem based on the [Breast Cancer Wisconsin Dataset](https://scikit-learn.org/stable/datasets/toy_dataset.html#breast-cancer-dataset).
The dataset contains 30 numeric features and the binary target class (benign/malignant).

The code illustrating the implementation of a machine learning pipeline in KDL is composed of three parts:

- Data preparation
- Traditional ML models (in scikit-learn)
- Neural network models (in PyTorch)

More information on each of these steps:

- **Data preparation**
  (code in [lab/processes/prepare_data/main.py](lab/processes/prepare_data/main.py)):
  the dataset is loaded from sklearn datasets and normalized;
  the transformed data are split into train, validation and test sets;
  and the processed data are stored on the shared volume.
- **Traditional ML models (in scikit-learn)**
  (code in [lab/processes/train_standard_classifiers/main.py](lab/processes/train_standard_classifiers/main.py)):
  the processed datasets are loaded from the shared volume as arrays;
  the script iterates through a number of classification algorithms,
  including logistic regression, naïve Bayes, random forest, gradient boosting, etc.;
  validation accuracy is computed and logged to MLflow.
- **Neural network models (in PyTorch)**
  (code in [lab/processes/train_dnn_pytorch/main.py](lab/processes/train_dnn_pytorch/main.py)):
  the processed datasets are loaded from the shared volume as torch DataLoaders;
  the script initiates a densely connected neural network for binary classification
  and launches its training and validation;
  the training history (accuracy and loss per epoch on both training and validation data) are stored as an artifact in MLflow (`training_history.csv` and visualized in `.png`).
  The model with the highest validation accuracy is saved as a .joblib file in MLflow artifacts, and is used to produce an assessment of model performance on the validation dataset (e.g. saving the loss and accuracy metrics, and the confusion matrix of the validation set, `confusion_matrix.png`, all logged to MLflow).

The execution of the example classification pipeline on Drone agents is specified in [.drone.yml](.drone.yml) 
(for simplicity, we are omitting various additional components here, such as the environment variables and the AWS secrets):

```yaml
---
kind: pipeline
type: kubernetes
name: example-pipeline

trigger:
  ref:
    - refs/tags/run-example-*
```

To **launch the execution** of this pipeline on Drone runners, push a tag containing the name matching the defined trigger to the remote repository.
In this case, the tag pattern is `run-example-*`,
therefore to launch the execution run the following commands in the Terminal:
`git tag run-example-v0 && git push origin run-example-v0`.
For more information and examples, see the section Launching experiment runs (Drone) below.

The **results of executions** are stored in MLflow.
In the example of training traditional ML models, we are only tracking one parameter (the name of the classifier)and one metric (the obtained validation accuracy). In the PyTorch neural network training example, we are tracking the same metric (validation accuracy) for comparisons, but a different set of hyperparameters, such as learning rate, batch size, number of epochs etc.
In a real-world project, you are likely to be tracking many more parameters and metrics of interest.
The connection to MLflow to log these parameters and metrics is established via the code in the [main.py](lab/processes/train_standard_classifiers/main.py) and with the environment variables in [.drone.yml](.drone.yml).
For more information on MLflow tracking, see the section "Logging experiment results (MLflow)" below.
To see the tracked experiments, visit the MLflow tool UI.

## Importing library functions

Reusable functions can be imported from the library (`lib` directory) to avoid code duplication and to permit a more organized structuring of the repository.

**In Jupyter:** 
To import library code in notebooks, you may need to add the `lab` directory to PYTHONPATH, for example as follows:

```python
import sys
from pathlib import Path

DIR_REPO = Path.cwd().parent.parent
DIR_LAB = DIR_REPO / "lab"

sys.path.append(str(DIR_LAB))

from lib.viz import plot_confusion_matrix
```

**In Vscode**: 
Imports from `lab` directory subdirectories are recognized correctly by code linters 
thanks to the defined `PYTHONPATH=lab` in the .env environment file. 
However, they are not recognized by the terminal, 
so in order to run code with imports from Vscode terminal, 
prepend your calls to Python scripts with `PYTHONPATH=lab` as follows: 
`PYTHONPATH=lab python {filename.py}`.

**On Drone:** 
To be able to run imports from the `lib` directory on Drone, you may add it to PYTHONPATH in .drone.yml as indicated:

```yaml
environment:
  PYTHONPATH: /drone/src/lab
```

`/drone/src` is the location on the Drone runner that the repository is cloned to, and `lab` is the name of the laboratory section of our repository which includes `lib`.
This then allows importing library functions directly from the Python script that is being executed on the runner, for instance:

```python
from lib.viz import plot_confusion_matrix
```

To see a working example, refer to the existing `application-examples` pipeline defined in .drone.yml
(the PyTorch example pipeline uses library imports in `processes/pytorch_example/main.py`).

## Launching experiment runs (Drone)

To enable full tracability and reproducibility, all executions that generate results or artifacts
(e.g. processed datasets, trained models, validation metrics, plots of model validation, etc.)
are run on Drone runners instead of the user's Jupyter or Vscode tools.

This way, any past execution can always be traced to the exact version of the code that was run (`VIEW SOURCE </>` in the UI of the Drone run)
and the runs can be reproduced with a click of the button in the UI of the Drone run (`RESTART`).

The event that launches a pipeline execution is defined by the trigger specified in .drone.yml.
An example is shown below:

```yaml
trigger:
  ref:
    - refs/tags/process-data-*
```

With this trigger in place, the pipeline will be executed on Drone agents whenever a tag matching the pattern specified in the trigger is pushed to the remote repository, for example:

```bash
git tag process-data-v0
git push origin process-data-v0
```

Note: If using an external repository (e.g. hosted on Github), a delay in synchronization between Gitea and the mirrored external repo may cause a delay in launching the pipeline on the Drone runners.
This delay can be overcome by manually forcing a synchronization of the repository in the Gitea UI Settings.

### Docker images for experiments & trainings

In the `drone.yml` file you can specify the image that is going to be used for each pipeline step.

```yml
steps:
  - name: prepare-data
    image: konstellation/kdl-py:3.9-1.1.0
  ...
```

There are two recommendations regarding which image to use:

1. Using an official runtime image. These images are used for running the KDL Usertools and have everything you need to run your code. If using one of these images take into account that the first thing you would need to do in the drone pipeline is to install your custom dependencies (`pipenv install`). You can find info about runtimes and their docker images inside KDL in the Usertools Settings section.
2. Using a custom image. For this case it is recommended to build a new layer on top of the official runtime images adding whatever you need to run your experiments/trainings.

## Logging experiment results (MLflow)

To compare various experiments, and to inspect the effect of the model hyperparameters on the results obtained, you can use MLflow experiment tracking. 
Experiment tracking with MLflow enables logging the parameters with which every run was executed and the metrics of interest, as well as any artifacts produced by the run.

The experiments are only tracked from the executions on Drone. 
In local test runs, mlflow tracking is disabled (through the use of a mock object replacing mlflow in the process code). 

The environment variables for connecting to MLflow server are provided in .drone.yml:

```yaml
environment:
  MLFLOW_URL: http://mlflow-server:5000
  MLFLOW_S3_ENDPOINT_URL: http://{{ ProjectID }}:9000
```

The use of MLflow for experiment tracking is illustrated by the scikit-learn example pipeline in [lab/processes/train_standard_classifiers/main.py](lab/processes/train_standard_classifiers/main.py).

```python
import mlflow

mlflow.set_tracking_uri(MLFLOW_URL)
mlflow.set_experiment(MLFLOW_EXPERIMENT)

with mlflow.start_run(run_name=MLFLOW_RUN_NAME, tags=MLFLOW_TAGS):

    # (... experiment code ...)

    # Log to MLflow
    mlflow.log_param("classifier", model_name)
    mlflow.log_metric("validation_accuracy", val_acc)
```

For more information on logging data to runs, see [MLflow documentation on logging](https://www.mlflow.org/docs/latest/tracking.html#logging-data-to-runs).

Whenever one script execution trains various models (e.g. in hyperparameter search, where a model is trained with many different combinations of hyperparameters), it is helpful to use nested runs. This way, the sub-runs will appear grouped under the parent run in the MLflow UI:

```python
import mlflow

with mlflow.start_run(run_name=MLFLOW_RUN_NAME, tags=MLFLOW_TAGS):

    # (... experiment setup code shared between subruns ... )

    with mlflow.start_run(run_name=SUBRUN_NAME, nested=True, tags=MLFLOW_TAGS):

        # (... model training code ...)

        mlflow.log_metric("classifier", model_name)
        mlflow.log_param("validation_accuracy", val_acc)
```

To compare the executions and vizualise the effect of logged parameters on the logged metrics,
you can select the runs you wish to compare in the MLflow UI, select "Compare" and add the desired parameters and metrics to the visualizations provided through the UI.
Alternatively, the results can also be queried with the MLflow API. For more information on the latter, see [MLflow documentation on querying runs](https://www.mlflow.org/docs/latest/tracking.html#querying-runs-programmatically).


## Testing

To run the automated tests, you have two options: via command line or via the Vscode UI.

### Running tests from command line
You can use the command `pytest` directly from the terminal line as follows: 
   ```
   PYTHONPATH=lab pytest -v                              # Run all tests (verbose)
   PYTHONPATH=lab pytest -v lab/processes/prepare_data   # Run only tests in prepare_data
   PYTHONPATH=lab pytest -v -m unittest                  # Run only unit tests
   PYTHONPATH=lab pytest -v -m integration               # Run only integration tests
   ```

You may add other optional pytest arguments as needed
(see [pytest usage documentation](https://docs.pytest.org/en/6.2.x/usage.html)).

### Running tests from Vscode UI 
It is also possible to run the tests using the Vscode user interface. 
To run all tests, select `Ctrl+Shift+P`, then search for `Python: Run All Tests`. 

You may **run tests individually** 
by clicking on the `Run Test` option next to the name of the test in the editor.
If this option does not appear next to the test, 
check that your file name and test name both include the string "test_" or "_test", 
then run `Ctrl+Shift+P` and search for `Python: Discover Tests`.

**Interactive debugging:** 
Unlike the command line option, the UI option also permits the use of the interactive debugging tool in Vscode.
- First, place breakpoints in your code (by placing a marker, clicking to the left of the code line number). 
- Next, select `Debug Test` next to the test (if launching individually), or `Ctrl+Shift+D` (`Python: Debug All Tests`).
- Select your test configuration "All tests" / "Integration tests" / "Unit tests" and click on the Run icon (these configurations can be edited in `.vscode/launch.json`)
- Use the Debug Console to explore the variables at the breakpoints, and the debug controls to pass between breakpoints

### Data for testing

Integration tests (and some unit tests) require the existence of a dataset to be able to run. 
This temporary dataset is provided to such tests 
through the use of a test fixture defined in `conftest.py`, 
and is eliminated by the same fixture after the test is executed.
The fixture is passed to any test as a function argument, 
as seen in the following example (from KDL Project Template):

```python
# In conftest.py

@pytest.fixture(name="temp_data_dir", scope="module")
def temporary_cancer_data_directory(dir_temp="temp"):
    """
    Pytest fixture for those tests that require a data directory containing the cancer dataset arrays.
    As part of setup, the fixture creates those arrays in the temporary location specified by dir_temp

    Keyword Arguments:
        dir_temp {str} -- Path where the files will be temporarily generated; the directory is cleared up after
            running the test (default: {"temp"})
    """

    # Setup:
    prepare_cancer_data(dir_output=dir_temp)

    yield dir_temp

    # Teardown:
    shutil.rmtree(dir_temp)


# In test file:

def test_load_data_splits_as_npy_array(self, temp_data_dir):
    """
    Test that data splits can be loaded as numpy arrays.
    Note: requires dir_temp populated with .npy files as generated by prepare_cancer_data, prepared by
    test fixture temp_cancer_data_dir (in conftest.py)
    """
    result = load_data_splits(dir_processed=temp_data_dir, as_type="array")
    for array in result:
        assert isinstance(array, np.ndarray)

```

In the example above, the fixture `temporary_cancer_data_directory` (abbreviated with the name `temp_data_dir`) 
defines what happens before and after executing a test that uses that fixture:
- Before running the test, this fixture runs through the setup code,
creating our dataset locally using `prepare_cancer_data`. 
- Next, the fixture yields the location of that directory to the test function. 
- After the test function has terminated running, the fixture executes the teardown code.

If we drop the `temp_data_dir` parameter from this test function, 
the test will run without the fixture, 
and will fail because the required data directory does not exist.

To learn more, see the documentation on [pytest fixtures](https://docs.pytest.org/en/6.2.x/fixture.html).


