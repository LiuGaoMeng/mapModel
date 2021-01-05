<template>
    <div>
        <Button>Default</Button>
    <Button type="primary">Primary</Button>
    <Button type="dashed">Dashed</Button>
    <Button type="text">Text</Button>
    <br><br>
    <Button type="info">Info</Button>
    <Button type="success">Success</Button>
    <Button type="warning">Warning</Button>
    </div>
    
</template>
<script>
export default {
    data(){
        return{
            theme1: 'light'
        }
    }
}
</script>