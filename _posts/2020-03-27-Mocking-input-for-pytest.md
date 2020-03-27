---
layout: post
title:  "Mocking Input for Pytest"
date:   2020-03-27 15:02:00 +0100
# categories: python pytest testing mocking
---
First, some context: I created a method that prompted the user for creating a configuration file. Y/N on whether to go through the steps, 
and then the values for the 3 configuations to set. Each of those were a `input(<prompt_here>)` call.

### Example
```
Config file not found. Would you like to create one now?  Y
What is your name?  Fiona Gibbons
What is the apikey to save? 12345-asbc
What is the name of the application connecting to? Rhino Pants
```

The code worked okay, but then I wanted to test the integrated functionality of the created config file. So I needed some way to mock input for the tests. 

After some time, I figured out a setup that worked, albeit not optimal.

### Working setup:
**conftest.py** 
```
@pytest.fixture(scope="function")
def mock_fill_in_config(monkeypatch):
    input_values = ['y',
                    'mock user',
                    '123456',
                    'Rhino Pants']

    def mock_input(arg):

        return input_values.pop(0)

    monkeypatch.setattr(builtins, 'input', mock_input)
	
@pytest.fixture(scope="function")
def no_config_file(monkeypatch, tmp_path):
    monkeypatch.chdir(tmp_path)
```

**test_function.py** 
```
def test_no_config_file_prompts_valid_info(mock_fill_in_config, capsys, no_config_file, config_file_data):
    parsed_args = parse_all_arguments([])
    out, err = capsys.readouterr()
    assert "Could not find config file 'user-config.ini'" in out
    assert "Config file 'user-config.ini' created" in out
    assert parsed_args['user'] == 'mock user'
    assert parsed_args['apikey'] == '123456'
    assert parsed_args['app'] == 'Rhino Pants'
```

I wanted to re-write it so that the `input values` could be defined in the test function itself, rather than creating a bunch of fixtures
for the various test cases (happy path, error paths missing various attributes)


### First rewrite - specifying the values inside the test method
I created an object MockInput that can hold the list of input values. This object also has a function that pops the first value from the list. I then set the monkeypatch
to mock the system input to call mock_input instead.

**conftest.py**
```
class MockInput:
    def __init__(self, input_list):
        self.input = input_list

    def mock_input(self, arg):
        return self.input.pop(0)
```

**test_function.py**
```
def test_prompts_and_creates_config_file_if_no_config_file(no_config_file, monkeypatch, capsys):
    input_values = ['y',
                    'mock user',
                    '123456',
                    'Rhino Pants']
    prompts = MockInput(input_values)
    monkeypatch.setattr(builtins, 'input', prompts.mock_input)

    parsed_args = parse_all_arguments([])
    out, err = capsys.readouterr()
    assert "Could not find config file 'user-config.ini'" in out
    assert "Config file 'user-config.ini' created" in out
    assert parsed_args['user'] == 'mock user'
    assert parsed_args['apikey'] == '123456'
    assert parsed_args['app'] == 'Rhino Pants'

```

Downside to this approach: having to specify monkeypatch in each test, which I felt was too much necessary "setup" required. I would like to have just the list specified for the test.


### Rewrite #2 - specifying the list as parameter to the test method
Created a fixture that dealt with the monkeypatching, that still called the MockInput class. Now I had the problem: how can I pass a list of strings defined in the test
to the fixture which is called before the test?
Solution: indirect parametrization

**conftest.py**
```
# Still the same
class MockInput:
    def __init__(self, input_list):
        self.input = input_list

    def mock_input(self, arg):
        return self.input.pop(0)


# New stuff
@pytest.fixture(scope="function")
def mock_input_fixture(monkeypatch, request):
    prompts = MockInput(request.param)
    monkeypatch.setattr(builtins, 'input', prompts.mock_input)
```

**test_function.py** 
```
# Passes the list of strings to the 'mock_input_fixture'

@pytest.mark.parametrize('mock_input_fixture', [['y', 'mock user', '123456', 'Rhino Pants']], indirect=True)
def test_prompts_and_creates_config_file_if_no_config_file(no_config_file, mock_input_fixture):
    parsed_args = parse_all_arguments([])
    assert parsed_args['user'] == 'mock user'
    assert parsed_args['apikey'] == '123456'
    assert parsed_args['app'] == 'Rhino Pants'
```