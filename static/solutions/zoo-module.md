# add the output to random_file/outputs.tf

```t
output "petname" {
  value = random_pet.filename.id
}
```

create a modul dir:

```bash
mkdir zoo_file_
```

Inside there (main.tf):

```t
variable "petnames" {
  type = list(string)
}
resource "local_file" "zoo" {
  filename = "zoo.txt"
  content  = join("\n", var.petnames)
}
```

main.tf of root should look like this:

```t
module "first" {
  source    = "./random_file"
  extension = "txt"
  size      = 1337
}

module "second" {
  source    = "./random_file"
  extension = "txt"
  size      = 42
}

module "zoo" {
  source   = "./zoo_file"
  petnames = [module.first.petname, module.second.petname]
}

output "filenames" {
  value = [
    module.first.filename,
    module.second.filename
  ]
}

output "zoo_file_content" {
  value = module.zoo
}
```
