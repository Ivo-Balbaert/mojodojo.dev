# Guides

<script>
  import pages from '@temp/pages'
  export default {
    computed: {
      guides() {
        return pages
          .filter(p => p.path.indexOf('/guides/general/') >= 0)
          .sort((a,b) => (a.title > b.title) ? 1 : ((b.title > a.title) ? -1 : 0));
      },
      modules() {
        return pages
          .filter(p => p.path.indexOf('/guides/modules/') >= 0)
          .sort((a,b) => (a.categories > b.categories) ? 1 : ((b.categories > a.categories) ? -1 : 0));
      },
      intro() {
        return pages
          .filter(p => p.path.indexOf('/guides/intro_to_mojo/') >= 0)
          .sort((a,b) => (a.title > b.title) ? 1 : ((b.title > a.title) ? -1 : 0));
      }
    }
  }
</script>


## General
General guides

<table>
  <tr>
    <th>Guide</th>
    <th>Description</th>
  </tr>
  <tr v-for="command in guides">
   <td><a :href="command.path">{{ command.title }}</a></td>
   <td style="white-space: pre-wrap;">{{ command.frontmatter.usage }}</td>
  </tr>
</table>


## Modules
Quick tutorials on how to use various builtins and standard library modules

<table>
  <tr>
    <th>Item</th>
    <th>Module</th>
    <th>Description</th>
  </tr>
  <tr v-for="command in modules">
   <td><a :href="command.path">{{ command.title }}</a></td>
   <td style="white-space: pre-wrap;">{{ command.frontmatter.categories }}</td>
   <td style="white-space: pre-wrap;">{{ command.frontmatter.usage }}</td>
  </tr>
</table>

## Intro to Mojo
_This is in very early stages and under heavy development_

<table>
  <tr>
    <th>Section</th>
    <th>Description</th>
  </tr>
  <tr v-for="command in intro">
   <td><a :href="command.path">{{ command.title }}</a></td>
   <td style="white-space: pre-wrap;">{{ command.frontmatter.usage }}</td>
  </tr>
</table>
