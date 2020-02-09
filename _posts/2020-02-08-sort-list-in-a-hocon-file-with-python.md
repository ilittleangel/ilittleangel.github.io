---
layout: post
title:  "Sort a list in a Hocon file with Python 3.6 or higher"
date:   2020-02-08 21:28:33 +0200
categories: python hocon
---

* [Overview](#overview)
* [Input Metadata](#input-metadata)
* [The python code](#the-python-code)
    - [The python code](#the-python-code)
    - [Prepare input file](#prepare-input-file)
    - [Sort the keys of each element of the original list](#sort-the-keys-of-each-element-of-the-original-list)
    - [Sort Hocon by key](#sort-hocon-by-key)
    - [main](#main)
* [Run the python](#run-the-python) 
  
  
## Overview

Imagine you have a HOCON file with a list of input metadata. This input data grow with time
and you want to version it with a CVS to see what is adding or deleting in that list. 

One aproach would be sort the list inside the Hocon file and make a commit.

This little post is about sort a list inside a Hocon file using Python 3.6 or higher. 
  
  
## Input Metadata

* The list we want to sort could be in a HOCON file that contains an unnamed list.
* Or we could also have a HOCON file that contains an `input_metadata` field that is the list itself.
* An other rare scenario could be when find a file that only have elements but not the list. This file
is not a valid HOCON but the format of each element does.

<table>
<tr>
<th>
With `input_metadata` key
</th>
<th>
With no `input_metadata` (unnamed list) 
</th>
<th>
Without list, only items (rare)
</th>
</tr>

<tr>

<td>
<pre>
hocon
{     
  input_metadata: [
    {
      id: "000003",
      name: "metadata3",
      attributes: {},
      type: "type1"
    }
    {
      id: "000001",
      name: "metadata1",
      attributes: {},
      type: "type1"
    }
    {
      id: "000002",
      name: "metadata2",
      attributes: {},
      type: "type2"
    }
  ]
}
</pre>
</td>

<td>
<pre>
hocon
[
    {
      id: "000003",
      name: "metadata3",
      attributes: {},
      type: "type1"
    }
    {
      id: "000001",
      name: "metadata1",
      attributes: {},
      type: "type1"
    }
    {
      id: "000002",
      name: "metadata3",
      attributes: {},
      type: "type2"
    }
]
</pre>
</td>

<td>
<pre>
hocon (not valid hocon)
{
  id: "000003",
  name: "metadata3",
  attributes: {},
  type: "type1"
}
{
  id: "000001",
  name: "metadata1",
  attributes: {},
  type: "type1"
}
{
  id: "000002",
  name: "metadata3",
  attributes: {},
  type: "type2"
}
</pre>
</td>

</tr>
</table>


These situations will be manage in the python script, [here](#prepare-input-file).
  
  
  
## The python code

### PyHocon

There is a Python package `PyHocon` that parse a file or a string with a HOCON format and 
convert into a `ConfigTree` object. Parsing HOCON files is so easy, we only have to use the factory
methods.

```python
ConfigFactory.parse_file(hocon_file_path)
ConfigFactory.parse_string(hocon_string)
``` 

And we get a dictionary object loaded in memory with which it is very useful for handling 
configurations properties, i.e `process_metadata` or whatever.

>Full documentation of this open source project on GitHub [chimpler/pyhocon](https://github.com/chimpler/pyhocon)
  
  
  
### Prepare input file

For this purpose, we have a method `prepare_input_file(file)`

```python
def prepare_input_file(file):
    with open(file, 'r') as f:
        content = f.read()
        if 'process_data' not in content:
            tmp = open(f'{file}.tmp', 'w')
            tmp.write('['.rstrip('\r\n') + '\n' + content + '\n' + ']'.rstrip('\r\n'))
            tmp.close()
            return ConfigFactory.parse_file(tmp.name)
        else:
            return ConfigFactory.parse_file(file)['input_metadata']
```

* This method return a `ConfigTree` object with just the list that we want to sort.
* Include the rare situation whether the file comes only with the items of the list but not the list itself.
In this case we have to add `[` at the beginnig of file and `]` at the end.
* Also include the normal situation when the list is contained in its own field `input_matadata`.
* Not included the situation when the file contains a unnamed list. But this case is so easy to manage.
Only have to parse the file without get the field or transform de file, only `ConfigFactory.parse_file(file)`
* Finally the return is as following:
```
ConfigTree(
    [('process_data', [
        ConfigTree([('id', '000003'), ('name', 'metadata3'), ('attributes', ConfigTree()), ('type', 'type1')]), 
        ConfigTree([('id', '000001'), ('name', 'metadata1'), ('attributes', ConfigTree()), ('type', 'type1')]), 
        ConfigTree([('id', '000002'), ('name', 'metadata2'), ('attributes', ConfigTree()), ('type', 'type2')])
    ])]
)
```
  
  
  
### Sort the keys of each element of the original list

* This method create a new ConfigTree per each element of the initial list.
* For each element, iterate again on the list of items, but these items have been sorted in alphabetical order or what is the same, in natural String order.

```python
def sort_keys(config):
    res = []
    for config_tree in config:
        new = ConfigTree()
        for k, v in sorted(config_tree.items()):
            new.put(key=k, value=v)
        res.append(new)
    return res
```
  
  
  
### Sort Hocon by key

* This method perform the sort by using a key of each configTree element.
* `sorted()` method in python iterate over a list and in this case we have a list of ConfigTree's.
* Then we have to create a new ConfigTree object and put again the `input_metadata` key, or not! 
this is my case. So as you wish.
* Finally, `PyHocon` has another utility `HOCONConverter` to convert inputs into file formats 
(json, hocon, yaml and properties), so we can write a new file with the desire output format. In our case
the same Hocon that we found in the input.

```python
def sort_hocon_by_key(file, key):
    sorted_hocon = sorted(prepare_input_file(file), key=lambda config_tree: str(config_tree[key]).lower())
    config = ConfigTree()
    config.put(key='input_metadata', value=sort_keys(sorted_hocon))
    output = HOCONConverter.to_hocon(config=config, indent=2, level=1).replace(' = ', ': ')
    with open(f'{file}.out', 'w') as f:
        f.write(output)
```

  
  
### main

* In the main section we only have to call the principal method and remove the `*.tmp` files to clear all.
* Another good point is to put everything in a `try-catch` block to capture any exception and print the error message.

```python
try:
    sort_hocon_by_key(sys.argv[1], sys.argv[2])
    clear()
    print('Done!')
except Exception as e:
    error(e)
```
  
  
  
### Run the python

```shell script
python3.8 sortHconByKey <inputHoconFile.conf> <sortKey>
```
  
  
  

[Github Gist](https://gist.github.com/ilittleangel/efd4d9f2e8df52323ba909fed095575c)
