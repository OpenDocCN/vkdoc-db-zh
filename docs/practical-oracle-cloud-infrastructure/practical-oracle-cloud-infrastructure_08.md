# data.tf
...
data "oci_core_images" "centos_image" {
  compartment_id = var.tenancy_ocid
  operating_system = "CentOS"
  operating_system_version = 7
}
...
