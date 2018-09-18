Machine Learning Versioning Tools - MLV-tools
=============================================
Public repository for versioning machine learning data

Keywords
--------

**Step metadata**: in this document it refers to the first code cell when it
is used to declare metadata such as parameters, dvc inputs/outputs, etc.

**Work directory**: the git top level directory of the project to version.
(If the project does not use git, which is not recommended, use --working-dir
 argument on each command call)


Tools
-----

**ipynb_to_python**: this command convert a given *Jupyter Notebook* to a
parameterized and executable *Python3 script* (see specific syntax in section below)

    ipynb_to_python -n [notebook_path] -o [python_script_path]
    
**gen_dvc**: this command create a dvc command which call the script generated by ipynb_to_python.  

    gen_dvc -i [python_script] --out-py-cmd [python_command] \
                  --out-bash-cmd [dvc_command]
    
Configuration
-------------

A configuration file can be provided, but it is not mandatory. 
It's default location is in the **working directory**, ie `[working_dir]/.mlvtools`. 
But it can be in a custom file provided as a command argument.

The configuration file format is JSON

    {
    "path": {
    	"python_script_root_dir": "[path_to_the_script_directory]",
    	"dvc_cmd_root_dir": "[path_to_the_dvc_cmd_directory]"
    	}
    "ignore_keys: ["keywords", "to", "ignore"],
    'dvc_var_python_cmd_path': 'MLV_PY_CMD_PATH_CUSTOM',
    'dvc_var_python_cmd_name': 'MLV_PY_CMD_NAME_CUSTOM',
    }

All given path must be relative to the **working directory**

- *path_to_the_script_directory*: is the directory where **Python 3** script will be generated using 
**ipynb_to_script** command. The **Python 3** script name is based on the notebook name.

        ipynb_to_script -n ./data/My\ Notebook.ipynb 
        
        Generated script: `[path_to_the_script_directory]/my_notebook.py`
        
- *path_to_the_dvc_cmd_directory*: is the directory where **DVC** commands will be generated using 
**gen_dvc** command. Generated command names are based on **Python 3** script name.

        gen_dvc -i ./scripts/my_notebook.py
        
        Generated commands: `[path_to_the_python_cmd_directory]/my_notebook_dvc`
                
- *ignore_keys*: list of keywords use to discard a cell. Default value is *['# No effect ]*.
    (See *Discard cell* section)
                          
- *dvc_var_python_cmd_path*, *dvc_var_python_cmd_name*, *dvc_var_meta_filename*: they allow to customize variable names which 
can be used in **dvc-cmd** Docstring parameter. They respectively correspond to the variables holding the python command 
file path, the file name and the variable holding the **DVC** default meta file name. Default values are 'MLV_PY_CMD_PATH',
 'MLV_PY_CMD_NAME' and 'MLV_DVC_META_FILENAME'. (See DVC Command/Complex cases section for usage) 


Jupyter Notebook syntax
-----------------------

The **Step metadata** cell is used to declare script parameters and **DVC** outputs and dependencies.
This can be done using basic Docstring syntax. This Docstring must be the first statement is this cell, only
comments can be writen above. 


### Good practices 

Avoid using relative paths in your Jupyter Notebook because they are relative to 
the notebook location which is not the same when it will be converted to a script.


### Parameterize

Parameter can be declared in the **Jupyter Notebook** using basic Docstring syntax.
This parameter description is used to generate configurable and executable python scripts.

Parameters declaration in **Jupyter Notebook**:

**Jupyter Notebook**: process_files.ipynb

    
    #:param [type]? [param_name]: [description]?
    """
    :param str input_file: the input file
    :param output_file: the output_file
    :param rate: the learning rate
    :param int retry:
    """
    
Generated **Python3 script**:

    [...]
    def process_file(input_file: str, output_file, rate, retry:int):
        """
         ...
        """
    [...]

Script command line parameters:

    my_script.py -h
    
    usage: my_cmd [-h] --input-file INPUT_FILE --output-file OUTPUT_FILE --rate
                 RATE --retry RETRY
    
    Command for script [script_name]
    
    optional arguments:
      -h, --help            show this help message and exit
      --input-file INPUT_FILE
                            the input file
      --output-file OUTPUT_FILE
                            the output_file
      --rate RATE           the rate
      --retry RETRY

All declared arguments are required.

### DVC command

A **DVC** command is a wrapper over **dvc run** command called on a **Python 3** script generated 
with **ipynb_to_python** command. It is a step of a pipeline. 

It is based on data declared in **notebook metadata**,
 2 modes are available:
    - describe only input/output for simple cases (recommended)
    - describe full command for complex cases

#### Simple cases

Syntax
    
    :param str input_csv_file: Path to input file
    :param str output_csv_file: Path to output file
    [...]
    
    [:dvc-[in|out][\s{related_param}]?:[\s{file_path}]?]*
    [:dvc-extra: {python_other_param}]?
    
    :dvc-in: ./data/filter.csv
    :dvc-in input_csv_file: ./data/info.csv    
    :dvc-out: ./data/train_set.csv    
    :dvc-out output_csv_file: ./data/test_set.csv
    :dvc-extra: --mode train --rate 12
       
Provided **{file_path}** path can be absolute or relative to the git top dir.

The **{related_param}** is a parameter of the corresponding **Python 3** script,
 it is filled in for the python script call

The **dvc-extra** allows to declare parameters which are not dvc outputs or dependencies.
Those parameters are provided to the call of the **Python 3** command.
 
    pushd $(git rev-parse --show-toplevel)
    
    INPUT_CSV_FILE="./data/info.csv"
    OUTPUT_CSV_FILE="./data/test_set.csv"
   
    dvc run \
    -d ./data/filter.csv\
    -d $INPUT_CSV_FILE\
    -o ./data/train_set.csv\
    -o $OUTPUT_CSV_FILE\
    gen_src/python_script.py --mode train --rate 12 
            --input-csv-file $INPUT_CSV_FILE 
            --output-csv-file $OUTPUT_CSV_FILE

        
    
#### Complex cases

Syntax
    
    :dvc-cmd: {dvc_command}

    :dvc-cmd: dvc run -o ./out_train.csv -o ./out_test.csv 
        "$MLV_PY_CMD_PATH -m train --out ./out_train.csv && 
         $MLV_PY_CMD_PATH -m test --out ./out_test.csv"
    
This syntax allows to provide the full dvc command to generate. All paths can be absolute or relative to the git top dir.
The variables $MLV_PY_CMD_PATH and $MLV_PY_CMD_NAME are available. They respectively contains the path and the name
 of the corresponding python command.
The variable $MLV_DVC_META_FILENAME contains the default name of the **DVC** meta file.
 
    pushd $(git rev-parse --show-toplevel)
    MLV_PY_CMD_PATH="gen_src/python_script.py"
    MLV_PY_CMD_NAME="python_script.py"
        
    dvc run -f $MLV_DVC_META_FILENAME -o ./out_train.csv \
        -o ./out_test.csv \
        "$MLV_PY_CMD_PATH -m train --out ./out_train.csv && \
        $MLV_PY_CMD_PATH -m test --out ./out_test.csv"    
    popd


### Discard cell

Some cells in **Jupyter Notebook** are executed only to watch intermediate results.
In a **Python 3** script those are statements with no effect. 
The comment **# No effect** allows to discard a whole cell content to avoid waste of 
time running those statements. It is possible to customize the list of discard keywords, see *Configuration* section.


Contributing
------------

We happily welcome contributions to MLV-tools. Please see our [contribution](./CONTRIBUTING.md) guide for details.